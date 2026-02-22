# 14 — Matrix Server Hosting for Agent Communication

## Question

How should we self-host a Matrix homeserver so that multiple OpenClaw AI agents (each with their own Matrix account) can communicate in shared rooms, with humans able to observe and participate?

## TL;DR

**Use Conduit** (Rust, lightweight) with Docker Compose for a private, non-federated Matrix homeserver. It runs on <512 MB RAM, has a single-binary deployment, and a simple `allow_federation = false` toggle. Create one Matrix account per agent, generate access tokens via the login API, and configure OpenClaw's multi-account Matrix plugin to bind each account to an agent. Disable E2EE for agent-only rooms (simpler key management, no device verification needed). Enable E2EE only if humans use clients like Beeper that require it. The whole setup fits in a single `docker-compose.yml` with ~20 lines of config.

---

## 1. Matrix Homeserver Options

### 1a. Synapse (Python/Twisted) — Reference Implementation

- **Market share:** ~85.5% of Matrix servers as of December 2025 ([TWIM 2025-12-19](https://matrix.org/blog/2025/12/19/this-week-in-matrix-2025-12-19/))
- **Maturity:** Most complete feature set, best-documented admin API
- **Resource usage:** 500 MB–1 GB RAM minimum; can spike to 2+ GB when joining large federated rooms; PostgreSQL recommended for production ([Understanding Synapse Hosting](https://matrix.org/docs/older/understanding-synapse-hosting/))
- **Scaling:** Supports worker-based horizontal scaling for large deployments
- **Storage:** Can consume significant disk space, especially with media and federated rooms; 20+ GB recommended even for small deployments

**Verdict for our use case:** Overkill. The Python runtime and PostgreSQL dependency add unnecessary overhead for a handful of bot accounts on a private server.

### 1b. Dendrite (Go) — Second Generation

- **Market share:** ~3.1% ([TWIM 2025-12-19](https://matrix.org/blog/2025/12/19/this-week-in-matrix-2025-12-19/))
- **Status:** **Maintenance mode** as of 2025 — security fixes only, no new features ([GitHub dendrite](https://github.com/element-hq/dendrite))
- **Resource usage:** Lower than Synapse (~200–500 MB RAM), single binary or polylith deployment
- **Database:** SQLite (single-user) or PostgreSQL (multi-user)

**Verdict for our use case:** Viable but risky. Maintenance mode means no bug fixes beyond security patches. Not recommended for new deployments.

### 1c. Conduit (Rust) — Lightweight

- **Market share:** ~4.0% ([TWIM 2025-12-19](https://matrix.org/blog/2025/12/19/this-week-in-matrix-2025-12-19/))
- **Resource usage:** Runs on devices with 2 GB RAM (tested on ODROID HC 2); typical usage <512 MB for small deployments ([Conduit docs](https://docs.conduit.rs/deploying/docker.html))
- **Database:** RocksDB (default, fast) or SQLite (simpler)
- **Scaling:** Single instance only, no horizontal scaling
- **Configuration:** Simple TOML file with `allow_federation = false` toggle
- **Docker image:** ~30 MB compressed

**Verdict for our use case: Best fit.** Minimal resources, simple config, Rust performance, single binary. The lack of horizontal scaling is irrelevant for 5–20 bot accounts.

### Summary Table

| | Synapse | Dendrite | Conduit |
|---|---|---|---|
| Language | Python | Go | Rust |
| Min RAM | ~512 MB | ~200 MB | ~100 MB |
| Recommended RAM | 1–2 GB | 512 MB | 256–512 MB |
| Database | PostgreSQL (recommended) | PostgreSQL/SQLite | RocksDB/SQLite |
| Federation disable | Workaround (empty whitelist) | Config flag | `allow_federation = false` |
| Status | Active (Element maintains) | Maintenance mode | Active |
| Docker image size | ~500 MB | ~100 MB | ~30 MB |
| Best for | Large/federated deployments | Mid-size, Go shops | Small/private servers |

---

## 2. Minimal Setup for Agent Use

### 2a. Docker Compose with Conduit

```yaml
# docker-compose.yml
services:
  conduit:
    image: matrixconduit/matrix-conduit:latest
    restart: unless-stopped
    ports:
      - "6167:6167"
    volumes:
      - conduit-data:/var/lib/matrix-conduit
    environment:
      CONDUIT_SERVER_NAME: "agents.internal"       # your domain or internal name
      CONDUIT_DATABASE_BACKEND: "rocksdb"
      CONDUIT_PORT: "6167"
      CONDUIT_MAX_REQUEST_SIZE: "20000000"          # 20 MB
      CONDUIT_ALLOW_REGISTRATION: "true"            # enable to create accounts, disable after
      CONDUIT_ALLOW_FEDERATION: "false"             # private server, no federation
      CONDUIT_TRUSTED_SERVERS: '["matrix.org"]'     # irrelevant with federation off
      CONDUIT_MAX_CONCURRENT_REQUESTS: "100"

volumes:
  conduit-data:
```

Start: `docker compose up -d`

The server is available at `http://localhost:6167`. For production, put a reverse proxy (Caddy, nginx, Traefik) in front for TLS.

**Source:** [Conduit Docker docs](https://docs.conduit.rs/deploying/docker.html)

### 2b. Alternative: Synapse with PostgreSQL

If you need Synapse for its admin API or feature completeness:

```yaml
# docker-compose.yml
services:
  synapse:
    image: matrixdotorg/synapse:latest
    restart: unless-stopped
    environment:
      SYNAPSE_CONFIG_PATH: /data/homeserver.yaml
    volumes:
      - synapse-data:/data
    ports:
      - "8008:8008"
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: changeme          # use a secret in production
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  synapse-data:
  postgres-data:
```

Generate initial config:

```bash
docker compose run --rm \
  -e SYNAPSE_SERVER_NAME=agents.internal \
  -e SYNAPSE_REPORT_STATS=no \
  synapse generate
```

Then edit `synapse-data/homeserver.yaml` to disable federation (see section 4).

**Source:** [Synapse Docker Compose](https://github.com/matrix-org/synapse/blob/develop/contrib/docker/docker-compose.yml), [Docker Hub synapse](https://hub.docker.com/r/matrixdotorg/synapse)

### 2c. Creating Bot/Agent Accounts

**With Conduit** (registration enabled):

```bash
# Register agent accounts via the Matrix client API
# Repeat for each agent (researcher, critic, orchestrator, etc.)

curl -XPOST "http://localhost:6167/_matrix/client/v3/register" \
  -H "Content-Type: application/json" \
  -d '{
    "auth": {"type": "m.login.dummy"},
    "username": "agent-researcher",
    "password": "secure-password-here",
    "inhibit_login": false
  }'
```

After creating all accounts, disable registration:

```bash
# Set CONDUIT_ALLOW_REGISTRATION=false and restart
docker compose down && docker compose up -d
```

**With Synapse** (admin CLI):

```bash
docker exec -it synapse register_new_matrix_user \
  -c /data/homeserver.yaml \
  -u agent-researcher \
  -p secure-password-here \
  --no-admin \
  http://localhost:8008
```

### 2d. Obtaining Access Tokens

Each OpenClaw agent needs an access token. Use the login API:

```bash
# Get access token for an agent account
curl -XPOST "http://localhost:6167/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "m.login.password",
    "identifier": {
      "type": "m.id.user",
      "user": "agent-researcher"
    },
    "password": "secure-password-here"
  }'

# Response includes: { "access_token": "syt_...", "device_id": "...", "user_id": "@agent-researcher:agents.internal" }
```

Save the `access_token` for OpenClaw configuration.

**Source:** `docs/channels/matrix.md:48-61` — OpenClaw Matrix login API example

### 2e. Creating Rooms Programmatically

```bash
# Create a shared agent room using any agent's access token
curl -XPOST "http://localhost:6167/_matrix/client/v3/createRoom" \
  -H "Authorization: Bearer syt_researcher_token_here" \
  -H "Content-Type: application/json" \
  -d '{
    "room_alias_name": "agent-workspace",
    "name": "Agent Workspace",
    "topic": "Multi-agent collaboration room",
    "visibility": "private",
    "invite": [
      "@agent-critic:agents.internal",
      "@human-observer:agents.internal"
    ],
    "creation_content": {
      "m.federate": false
    }
  }'
```

**Source:** [Matrix Client-Server API — createRoom](https://matrix.org/docs/older/client-server-api/)

### 2f. OpenClaw Configuration for Multi-Agent Matrix

This is the full OpenClaw config to connect multiple agents to the same Matrix homeserver:

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
      homeserver: "http://localhost:6167",  // or https://matrix.yourdomain.com
      groupPolicy: "allowlist",
      groups: {
        "#agent-workspace:agents.internal": {
          allow: true,
          requireMention: true,  // prevent infinite loops between agents
        },
      },
      accounts: {
        researcher: {
          accessToken: "syt_researcher_***",
        },
        critic: {
          accessToken: "syt_critic_***",
        },
      },
    },
  },
}
```

**Source:** `docs/channels/matrix.md` — multi-account section; `exploration/agents/agent-to-agent.md` — section 3 (channel-based architecture)

**Critical: Set `requireMention: true`** to prevent infinite agent reply loops. Each agent should only respond when explicitly @mentioned. See `exploration/agents/agent-to-agent.md` — section 7, caveat 2.

---

## 3. Encryption (E2EE)

### Should We Enable E2EE for Agent Rooms?

**Recommendation: No, not for agent-only rooms.**

| Factor | E2EE Enabled | E2EE Disabled |
|---|---|---|
| Setup complexity | High (device verification per agent) | None |
| Key management | Crypto state per account+token in SQLite | N/A |
| Agent restarts | May require re-verification | Transparent |
| Human observation | Works with Element, Beeper | Works with any client |
| Beeper compatibility | Required | Not compatible |
| Security benefit | Encrypted at rest on server | Messages stored in plaintext on server |
| Debugging | Harder (can't inspect server-side) | Easy (read messages from DB/admin API) |

For a **private, non-federated server** that you control, E2EE adds operational complexity without meaningful security benefit — you already control the server. The main reason to enable E2EE is if humans use **Beeper** (which requires it).

### How OpenClaw Handles E2EE

OpenClaw's Matrix plugin uses `@matrix-org/matrix-sdk-crypto-nodejs` (Rust crypto SDK):

- Crypto state stored per account+token in `~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/crypto/` (SQLite)
- Device verification required on first connection — must approve in another Matrix client
- If crypto module fails to load, E2EE silently degrades (logged as warning)
- Each agent account needs its own device verification

**Source:** `docs/channels/matrix.md:111-132` — Encryption section

### If You Do Enable E2EE

```json5
{
  channels: {
    matrix: {
      encryption: true,  // global default for all accounts
      accounts: {
        researcher: {
          accessToken: "syt_researcher_***",
          encryption: true,
        },
      },
    },
  },
}
```

After startup, verify each agent's device from an Element session logged into the same homeserver.

---

## 4. Federation

### Do We Need It?

**No.** For internal agent-to-agent communication on a single homeserver, federation adds:

- Attack surface (inbound federation listener)
- Resource consumption (state resolution for remote rooms)
- Complexity (TLS certificates, SRV records, well-known files)

Disable it.

### How to Disable Federation

**Conduit** (simplest):

```toml
# conduit.toml or environment variable
[global]
allow_federation = false
```

Or via Docker environment: `CONDUIT_ALLOW_FEDERATION=false`

**Source:** [Conduit Configuration](https://docs.conduit.rs/configuration.html)

**Synapse** (workaround — no single toggle):

```yaml
# homeserver.yaml

# 1. Empty federation whitelist blocks all outbound federation
federation_domain_whitelist: []

# 2. Remove or comment out the federation listener
# listeners:
#   - port: 8448
#     type: http
#     resources:
#       - names: [federation]

# 3. Disable federation in room creation defaults
default_room_version: "11"
```

**Source:** [Synapse issue #6395](https://github.com/matrix-org/synapse/issues/6395), [Synapse Configuration Manual](https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html)

---

## 5. Lightweight Alternatives

### 5a. Hosted Matrix Services

| Service | Pricing | Notes |
|---|---|---|
| **matrix.org** (free) | Free | Public homeserver, not private; accounts at @user:matrix.org |
| **Element Server Suite (ESS)** | Enterprise pricing (contact sales) | Managed Synapse Pro; overkill for bot accounts |
| **Beeper** | Consumer-focused | Bridge-oriented; not designed for bot hosting |
| **Etke.cc** | From ~5 EUR/month | Managed Matrix hosting, supports custom homeservers |

**Source:** [Element pricing](https://element.io/en/pricing), [Matrix hosting directory](https://matrix.org/ecosystem/hosting/)

**Verdict:** Hosted services make sense if you want zero maintenance. For agent-to-agent communication, creating accounts on the free **matrix.org** homeserver works for experimentation but is not private. For production, self-hosting with Conduit is simpler and cheaper than managed services.

### 5b. Non-Matrix Alternatives

If you only need agent chat rooms without the full Matrix protocol:

| Alternative | Pros | Cons |
|---|---|---|
| **`sessions_send` (A2A)** | No external service, low latency, full persona | Max 5 ping-pong turns, not human-observable |
| **Telegram groups** | Familiar, free, easy bot API | External dependency, not self-hosted |
| **Discord** | Rich API, bot support | External dependency, rate limits |
| **IRC (with ZNC)** | Ultra-lightweight, self-hosted | No E2EE, no media, limited features |
| **NATS/Redis pub/sub** | Fastest, fully self-hosted | No chat UI for human observation |

**Verdict:** Matrix is the best fit when you want: (a) self-hosted, (b) human-observable chat rooms, (c) multiple agent accounts, and (d) optional E2EE. If you only need programmatic A2A without human observation, use OpenClaw's built-in `sessions_send` instead (`exploration/agents/agent-to-agent.md` — section 2).

---

## 6. Practical Considerations

### 6a. Backup

**Conduit:**

```bash
# Backup the RocksDB data volume
docker compose stop conduit
docker run --rm -v conduit-data:/data -v $(pwd)/backups:/backup \
  alpine tar czf /backup/conduit-$(date +%Y%m%d).tar.gz /data
docker compose start conduit
```

**Synapse:** Back up both the PostgreSQL database and the media store:

```bash
docker exec db pg_dump -U synapse synapse > backups/synapse-$(date +%Y%m%d).sql
```

### 6b. Monitoring

- **Conduit** exposes no Prometheus metrics natively. Monitor via Docker health checks and log tailing.
- **Synapse** has built-in Prometheus metrics (`enable_metrics: true` in homeserver.yaml).

Minimal monitoring for a private agent server:

```yaml
# docker-compose.yml addition
services:
  conduit:
    # ... existing config ...
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6167/_matrix/client/versions"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### 6c. Security Hardening

For a private, non-federated server:

1. **Disable registration after account creation** — `CONDUIT_ALLOW_REGISTRATION=false`
2. **Bind to localhost only** — put a reverse proxy in front; do not expose port 6167 directly
3. **Use strong passwords** for agent accounts (or use access tokens only)
4. **Network isolation** — run on a private network (e.g., Tailscale) so only your OpenClaw gateway can reach it
5. **No public .well-known** — since federation is off, no need to expose `.well-known/matrix/server`
6. **Rate limiting** — Conduit has built-in rate limiting; Synapse has `rc_message` config

```yaml
# Production docker-compose.yml with reverse proxy
services:
  conduit:
    image: matrixconduit/matrix-conduit:latest
    restart: unless-stopped
    # Do NOT expose ports directly — use reverse proxy
    expose:
      - "6167"
    volumes:
      - conduit-data:/var/lib/matrix-conduit
    environment:
      CONDUIT_SERVER_NAME: "agents.internal"
      CONDUIT_DATABASE_BACKEND: "rocksdb"
      CONDUIT_PORT: "6167"
      CONDUIT_ALLOW_REGISTRATION: "false"
      CONDUIT_ALLOW_FEDERATION: "false"
      CONDUIT_MAX_REQUEST_SIZE: "20000000"
      CONDUIT_MAX_CONCURRENT_REQUESTS: "100"
    networks:
      - matrix-internal

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data
    networks:
      - matrix-internal

networks:
  matrix-internal:
    driver: bridge

volumes:
  conduit-data:
  caddy-data:
```

With Caddyfile:

```
matrix.yourdomain.com {
    reverse_proxy conduit:6167
}
```

### 6d. Maintenance

- **Updates:** Pull new Conduit image periodically: `docker compose pull && docker compose up -d`
- **Log rotation:** Conduit logs to stdout; Docker handles rotation via `--log-opt max-size=10m`
- **Storage growth:** Minimal for agent-only rooms (text messages are tiny). Monitor with `docker system df`.

---

## 7. Recommended Setup Sequence

1. Deploy Conduit with Docker Compose (section 2a)
2. Create agent accounts with registration enabled (section 2c)
3. Disable registration (set `CONDUIT_ALLOW_REGISTRATION=false`, restart)
4. Obtain access tokens for each agent (section 2d)
5. Create shared rooms (section 2e)
6. Configure OpenClaw multi-account Matrix (section 2f)
7. Set `requireMention: true` to prevent agent reply loops
8. Test with a human account in Element joining the same room
9. Harden with reverse proxy and private networking (section 6c)

---

## References

### OpenClaw Source & Docs

| Reference | Location |
|---|---|
| Matrix channel plugin docs | `docs/channels/matrix.md` |
| Multi-account config | `docs/channels/matrix.md:139-178` |
| E2EE setup | `docs/channels/matrix.md:111-132` |
| Agent-to-agent via Matrix | `exploration/agents/agent-to-agent.md` — section 3 |
| Reply loop risk | `exploration/agents/agent-to-agent.md` — section 7, caveat 2 |
| Channel config schema | `src/config/zod-schema.providers.ts:41` (passthrough for extension channels) |
| Multi-agent bindings | `docs/concepts/multi-agent.md` |

### External Sources

| Source | URL |
|---|---|
| Matrix server comparison | [matrixdocs.github.io/docs/servers/comparison](https://matrixdocs.github.io/docs/servers/comparison) |
| Matrix server ecosystem | [matrix.org/ecosystem/servers](https://matrix.org/ecosystem/servers/) |
| Conduit Docker deployment | [docs.conduit.rs/deploying/docker.html](https://docs.conduit.rs/deploying/docker.html) |
| Conduit configuration reference | [docs.conduit.rs/configuration.html](https://docs.conduit.rs/configuration.html) |
| Synapse Docker Compose | [github.com/matrix-org/synapse/.../docker-compose.yml](https://github.com/matrix-org/synapse/blob/develop/contrib/docker/docker-compose.yml) |
| Synapse configuration manual | [element-hq.github.io/synapse/.../config_documentation.html](https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html) |
| Synapse federation disable | [github.com/matrix-org/synapse/issues/6395](https://github.com/matrix-org/synapse/issues/6395) |
| Matrix Client-Server API | [matrix.org/docs/older/client-server-api](https://matrix.org/docs/older/client-server-api/) |
| Matrix hosting directory | [matrix.org/ecosystem/hosting](https://matrix.org/ecosystem/hosting/) |
| Element pricing | [element.io/en/pricing](https://element.io/en/pricing) |
| Self-hosting Matrix in 2025 | [blog.klein.ruhr/self-hosting-matrix-in-2025](https://blog.klein.ruhr/self-hosting-matrix-in-2025) |
| TWIM Dec 2025 (server stats) | [matrix.org/blog/2025/12/19/this-week-in-matrix-2025-12-19](https://matrix.org/blog/2025/12/19/this-week-in-matrix-2025-12-19/) |
