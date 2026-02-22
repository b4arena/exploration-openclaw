# External Resources for OpenClaw

> Compiled: February 2026. Links should be verified before relying on them — the OpenClaw ecosystem moves fast and URLs may change or become outdated.

OpenClaw (formerly Clawdbot, then Moltbot) is a personal AI assistant / multi-channel AI gateway created by Peter Steinberger. It launched November 2025, went viral in late January 2026, and was rebranded twice due to Anthropic trademark disputes. As of mid-February 2026, the creator announced joining OpenAI and transferring the project to an open-source foundation.

---

## Technical / Deployment

### Official Documentation

| Title | URL | Description |
|---|---|---|
| OpenClaw Docs — Getting Started | https://docs.openclaw.ai/start/getting-started | Official onboarding guide, CLI wizard walkthrough |
| OpenClaw Docs — Docker | https://docs.openclaw.ai/install/docker | Official Docker deployment documentation |
| OpenClaw Docs — Nix | https://docs.openclaw.ai/install/nix | Official Nix/NixOS installation guide |
| OpenClaw Docs — Security | https://docs.openclaw.ai/gateway/security | Gateway security model, sandbox, permissions |
| DeepWiki — Architecture Deep Dive | https://deepwiki.com/openclaw/openclaw/15.1-architecture-deep-dive | Auto-generated architecture documentation from source |

### Setup Guides & Tutorials

| Title | URL | Description |
|---|---|---|
| Codecademy — Installation to First Chat Setup | https://www.codecademy.com/article/open-claw-tutorial-installation-to-first-chat-setup | Step-by-step beginner tutorial from install to first conversation |
| freeCodeCamp — Full Tutorial for Beginners | https://www.freecodecamp.org/news/openclaw-full-tutorial-for-beginners/ | Comprehensive beginner-friendly tutorial with video |
| Runcell — Self-Hosted AI Assistant You Can Message From Anywhere | https://www.runcell.dev/blog/deploy-openclaw | Deployment guide for self-hosting |
| DigitalOcean — How to Run OpenClaw | https://www.digitalocean.com/community/tutorials/how-to-run-openclaw | Cloud deployment tutorial on DigitalOcean |
| AIML API — Run OpenClaw in Docker: Secure Local Setup | https://aimlapi.com/blog/running-openclaw-in-docker-secure-local-setup-and-practical-workflow-guide | Docker setup with security focus |
| Ajeet Raina — Running OpenClaw Using Docker Desktop | https://www.ajeetraina.com/running-openclaw-using-docker-desktop-your-personal-ai-assistant-in-a-container/ | Docker Desktop specific walkthrough |
| Simon Willison — Running OpenClaw in Docker | https://til.simonwillison.net/llms/openclaw-docker | Concise TIL-style Docker setup notes |
| Lilys AI — Install on Linux (Ubuntu 24.04) with Telegram/WhatsApp | https://lilys.ai/en/notes/openclaw-20260210/install-openclaw-linux-telegram-whatsapp | Linux-specific install with channel integration |

### Kubernetes & Helm

| Title | URL | Description |
|---|---|---|
| openclaw-helm (Helm chart) | https://github.com/serhanekicii/openclaw-helm | Community Helm chart for Kubernetes deployment |
| OpenClaw.rocks — Deploy on Kubernetes | https://openclaw.rocks/blog/deploy-openclaw-kubernetes | Step-by-step K8s deployment guide |
| Unraid Forum — OpenClaw Support Thread | https://forums.unraid.net/topic/196865-support-openclaw-ai-personal-assistant/ | Community support for Unraid container deployment |

### NixOS

| Title | URL | Description |
|---|---|---|
| nix-openclaw (official) | https://github.com/openclaw/nix-openclaw | Official Nix flake: gateway + tools, launchd/systemd service |
| Scout-DJ/openclaw-nix | https://github.com/Scout-DJ/openclaw-nix | Hardened NixOS flake with 2-line config |
| clawdinators | https://github.com/openclaw/clawdinators | Declarative infra + NixOS modules for CLAWTINATOR hosts |
| Bogdan Buduroiu — microvm.nix Sandbox | https://buduroiu.com/blog/openclaw-microvm/ | Running OpenClaw in a microvm.nix sandbox for isolation |
| Stacker News — NixOS User's Guide | https://stacker.news/items/1423749 | Community guide for NixOS users |

### Architecture & Performance

| Title | URL | Description |
|---|---|---|
| SitePoint — Production Guide: 4 Weeks of Lessons | https://www.sitepoint.com/openclaw-production-lessons-4-weeks-self-hosted-ai/ | Real-world production experience: memory, scaling, gotchas |
| Rentier Digital (Medium) — Architecture That Actually Scales | https://medium.com/@rentierdigital/the-complete-openclaw-architecture-that-actually-scales-memory-cron-jobs-dashboard-and-the-c96e00ab3f35 | Memory, cron jobs, dashboard, common mistakes |
| Easton Dev — Three-Layer Design Deep Dive | https://eastondev.com/blog/en/posts/ai/20250205-openclaw-architecture-guide/ | Technical principles and extension practices |
| Snowan — Memory System Deep Dive | https://snowan.gitbook.io/study-notes/ai-blogs/openclaw-memory-system-deep-dive | Embeddings, vector search, hybrid search internals |
| Trilogy AI (Substack) — Deep Dive OpenClaw | https://trilogyai.substack.com/p/deep-dive-openclaw | Architecture overview with diagrams |
| Paolo (Substack) — Architecture Explained | https://ppaolo.substack.com/p/openclaw-system-architecture-overview | How the Gateway, agents, and tools interact |
| HackMD — Architecture Deep Dive (02/08/2026) | https://hackmd.io/Z39YLHZoTxa7YLu_PmEkiA | Community-maintained architecture notes |

---

## UX / End-User

### Channel Integration Guides

| Title | URL | Description |
|---|---|---|
| OpenClaw Docs — WhatsApp | https://docs.openclaw.ai/channels/whatsapp | Official WhatsApp channel setup |
| MarkTechPost — Getting Started with WhatsApp | https://www.marktechpost.com/2026/02/14/getting-started-with-openclaw-and-connecting-it-with-whatsapp/ | WhatsApp connection walkthrough |
| Digital Applied — WhatsApp Integration & Automation | https://www.digitalapplied.com/blog/openclaw-whatsapp-integration-messaging-automation-guide | WhatsApp automation patterns |
| LumaDock — WhatsApp Production Setup | https://lumadock.com/tutorials/openclaw-whatsapp-production-setup | Security + phone number best practices for WhatsApp |
| TechLatest (Medium) — Telegram + OpenClaw | https://medium.com/@techlatest.net/from-chat-app-to-ai-powerhouse-telegram-openclaw-151462ba0fc8 | Telegram integration deep dive |
| openclaw-ai.online — Channel Setup Tutorial | https://openclaw-ai.online/tutorials/setup-channel/ | Generic channel setup (Telegram, WhatsApp, Discord) |

### UI & Canvas

| Title | URL | Description |
|---|---|---|
| OpenClaw Docs — Canvas | https://docs.openclaw.ai/platforms/mac/canvas | Official Canvas documentation (HTML/CSS/JS workspace) |
| ClawUI (GitHub) | https://github.com/Kt-L/clawUI | Third-party desktop client (React + Vite + Electron) with rich UI |
| OpenClawLab — Web (Gateway) | https://openclawlab.com/docs/web/ | WebChat and Gateway UI documentation |

### Comparison Articles

| Title | URL | Description |
|---|---|---|
| DigitalOcean — What is OpenClaw? | https://www.digitalocean.com/resources/articles/what-is-openclaw | Overview + positioning vs other tools |
| CyberNews — OpenClaw Review 2026 | https://cybernews.com/ai-tools/openclaw-review/ | Independent review with pros/cons |
| CodeConductor — Top OpenClaw Alternatives | https://codeconductor.ai/blog/openclaw-alternatives/ | Comparison with competing platforms |
| eesel.ai — 5 Best OpenClaw Alternatives | https://www.eesel.ai/blog/openclaw-ai-alternatives | Side-by-side alternative analysis |

### Prompt Engineering & Workflow

| Title | URL | Description |
|---|---|---|
| Master OpenClaw in 30 Minutes | https://creatoreconomy.so/p/master-openclaw-in-30-minutes-full-tutorial | Setup + 5 real use cases + memory management |
| Geeky Gadgets — Beginners Guide: Timers, Webhooks, Loops | https://www.geeky-gadgets.com/openclaw-explained-beginners-guide-2026/ | Timers, webhooks, markdown, continuous loops |
| Mikhail Shcheglov (Substack) — Ultimate Guide | https://corpwaters.substack.com/p/the-ultimate-guide-to-openclaw | Comprehensive workflow guide |

---

## Use Case / Community

### Awesome Lists

| Title | URL | Description |
|---|---|---|
| awesome-openclaw (SamurAIGPT) | https://github.com/SamurAIGPT/awesome-openclaw | Curated list of resources, tools, skills, tutorials, articles |
| awesome-openclaw-skills (VoltAgent) | https://github.com/VoltAgent/awesome-openclaw-skills | 3000+ community-built skills organized by category |
| awesome-openclaw-usecases (hesamsheikh) | https://github.com/hesamsheikh/awesome-openclaw-usecases | Real-world use cases from tweets, blogs, community |

### Community Platforms

| Platform | URL | Description |
|---|---|---|
| Discord | https://discord.gg/clawd | Official Discord community |
| GitHub Discussions | https://github.com/openclaw/openclaw/discussions | Official Q&A and discussion forum |
| X (Twitter) Community | https://x.com/i/communities/2013441068562325602 | OpenClaw community on X |
| OpenClaw Shoutouts | https://openclaw.ai/shoutouts | Community showcase on official site |
| DeployClaw — Use Cases | https://deployclaw.com/tools/openclaw-use-cases | Curated real-world examples |

### Blog Posts & Articles

| Title | URL | Description |
|---|---|---|
| Bill Wang (Medium) — Comprehensive Guide | https://medium.com/@ozbillwang/understanding-openclaw-a-comprehensive-guide-to-the-multi-channel-ai-gateway-ad8857cd1121 | Multi-channel gateway deep dive |
| Enclave AI — 2026 Guide | https://enclaveai.app/blog/2026/02/14/openclaw-personal-ai-assistant-guide/ | Overview with privacy focus |
| Valletta Software — Architecture, Setup, Skills, Security | https://vallettasoftware.com/blog/post/openclaw-2026-guide | End-to-end guide covering all major topics |
| NxCode — Complete Guide (Clawdbot to OpenClaw) | https://www.nxcode.io/resources/news/openclaw-complete-guide-2026 | Full history and technical guide |
| ChatBench — 14 Must-Know Insights | https://www.chatbench.org/openclaw/ | Quick-hit insights for newcomers |
| Milvus Blog — Complete Guide to the Autonomous AI Agent | https://milvus.io/blog/openclaw-formerly-clawdbot-moltbot-explained-a-complete-guide-to-the-autonomous-ai-agent.md | Comprehensive technical explainer |
| TypeVar — Mastering the Lobster Way | https://typevar.dev/articles/openclaw/openclaw | Technical walkthrough of the framework |

### Podcasts & Interviews

| Title | URL | Description |
|---|---|---|
| Lex Fridman Podcast #491 — Peter Steinberger | https://lexfridman.com/peter-steinberger-transcript/ | In-depth interview with the creator (Feb 12, 2026) |
| IBM Security Intelligence — OpenClaw & Agent Security | https://www.ibm.com/think/podcasts/security-intelligence/openclaw-claude-opus-4-6-ai-agent-security | Security implications of AI agents |
| IBM Mixture of Experts — Codex Launch & OpenClaw Chaos | https://www.ibm.com/think/podcasts/mixture-of-experts/codex-launch-openclaw-moltbook-chaos-ai-agents | Industry context and agent ecosystem |
| Peter Steinberger's Blog — OpenClaw, OpenAI, and the Future | https://steipete.me/posts/2026/openclaw | Creator's own reflections on the project's future |

### Project Use Cases

| Title | URL | Description |
|---|---|---|
| DataCamp — 9 OpenClaw Projects to Build | https://www.datacamp.com/blog/openclaw-projects | From Reddit bots to self-healing servers |
| PsyPost — Psychological Complexity of Human-Computer Interaction | https://www.psypost.org/viral-ai-agent-openclaw-highlights-the-psychological-complexity-of-human-computer-interaction/ | Research perspective on AI agent UX |

### History & Context

| Title | URL | Description |
|---|---|---|
| Wikipedia — OpenClaw | https://en.wikipedia.org/wiki/OpenClaw | Encyclopedia article with full history |
| CNBC — From Clawdbot to OpenClaw | https://www.cnbc.com/2026/02/02/openclaw-open-source-ai-agent-rise-controversy-clawdbot-moltbot-moltbook.html | Rise, controversy, and triple rebrand |
| LumaDock — Why the Name Changed Twice | https://lumadock.com/blog/clawdbot-moltbot-openclaw-rebrand | Rebrand timeline and rationale |
| Fortune — OpenAI's OpenClaw Hire | https://fortune.com/2026/02/17/what-openais-openclaw-hire-says-about-the-future-of-ai-agents/ | Industry analysis of the OpenAI acquisition |
| Decrypt — Will It Stay Open Source? | https://decrypt.co/358129/openclaw-creator-offers-acquire-ai-sensation-stay-open-source | Open-source future discussion |

---

## Plugin / Extension Ecosystem

### Skills & ClawHub

| Title | URL | Description |
|---|---|---|
| OpenClaw Docs — Skills | https://docs.openclaw.ai/tools/skills | Official skills documentation (SKILL.md format) |
| ClawHub (GitHub) | https://github.com/openclaw/clawhub | Official skill directory / marketplace |
| Zen van Riel — Custom Skill Creation Step by Step | https://zenvanriel.nl/ai-engineer-blog/openclaw-custom-skill-creation-guide/ | Practical guide to writing your own skill |
| Easton Dev — Image Processing Skill from Scratch | https://eastondev.com/blog/en/posts/ai/20250205-openclaw-skill-tutorial/ | Complete guide building a specific skill type |
| LumaDock — Skills Guide | https://lumadock.com/tutorials/openclaw-skills-guide | What skills are and how to use them |
| openclawskill.cc — Create, Template, Test, Rollout | https://openclawskill.cc/blog/how-to-create-an-openclaw-skill | Skill development lifecycle |
| Nwosu Rosemary (Medium) — Setting Up Skills | https://nwosunneoma.medium.com/setting-up-skills-in-openclaw-d043b76303be | Beginner's perspective on skill setup |
| Apiyi — 50+ Official Integrations | https://help.apiyi.com/en/openclaw-extensions-ecosystem-guide-en.html | Overview of the official extension ecosystem |

### MCP (Model Context Protocol)

| Title | URL | Description |
|---|---|---|
| SafeClaw — MCP Integration Guide | https://safeclaw.io/blog/openclaw-mcp | How to use MCP with OpenClaw |
| openclaw-mcp (GitHub) | https://github.com/freema/openclaw-mcp | MCP server bridge between Claude.ai and OpenClaw |
| openclaw-mcp-adapter (GitHub) | https://github.com/androidStern-personal/openclaw-mcp-adapter | Plugin exposing MCP server tools as native agent tools |
| openclaw-mcp-plugin (GitHub) | https://github.com/lunarpulse/openclaw-mcp-plugin | MCP plugin with remote streamable HTTP |
| GitHub Issue #4834 — Native MCP Support | https://github.com/openclaw/openclaw/issues/4834 | Feature request / discussion for native MCP |
| mcp-hub skill | https://playbooks.com/skills/openclaw/skills/mcp-hub | Skill providing access to 1200+ MCP servers |

### Security Advisories (Ecosystem)

| Title | URL | Description |
|---|---|---|
| CyberPress — ClawHavoc: 1,184 Malicious Skills | https://cyberpress.org/clawhavoc-poisons-openclaws-clawhub-with-1184-malicious-skills/ | Supply chain attack on ClawHub marketplace |
| Security Boulevard — Securing OpenClaw Against ClawHavoc | https://securityboulevard.com/2026/02/securing-openclaw-againstclawhavoc/ | Mitigation strategies for the supply chain attack |
| Microsoft Security Blog — Running OpenClaw Safely | https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/ | Identity, isolation, and runtime risk guidance |
| Cisco Blog — Personal AI Agents Are a Security Nightmare | https://blogs.cisco.com/ai/personal-ai-agents-like-openclaw-are-a-security-nightmare | Broad security analysis of agent architectures |
| Barrack.ai — Security Vulnerabilities & Safe Setup | https://blog.barrack.ai/openclaw-security-vulnerabilities-2026/ | Vulnerability overview with safe deployment recommendations |
| SecurityWeek — SecureClaw Open Source Tool | https://www.securityweek.com/openclaw-security-issues-continue-as-secureclaw-open-source-tool-debuts/ | Community security scanning tool for OpenClaw |

---

## Key Official Links

| Resource | URL |
|---|---|
| GitHub Repository | https://github.com/openclaw/openclaw |
| Official Website | https://openclaw.ai |
| Documentation | https://docs.openclaw.ai |
| npm Package | https://www.npmjs.com/package/openclaw |
| Discord | https://discord.gg/clawd |
| DeepWiki | https://deepwiki.com/openclaw/openclaw |
| Releases | https://github.com/openclaw/openclaw/releases |
