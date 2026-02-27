# Podman Migration — Docker to Podman for OpenClaw Sandboxing

Research into whether OpenClaw's sandboxing works with Podman containers, and what a migration from Docker to Podman entails.

## Current State

- **Mimas runs Docker CE** for both the OpenClaw gateway and agent sandbox containers
- The forge explicitly documents Docker usage (`forge/CLAUDE.md`: "OpenClaw uses Docker, not Podman")
- Sandbox containers are managed via `docker create`, `docker start`, `docker exec` commands in `src/agents/sandbox/docker.ts`
- Custom sandbox images are built with standard Dockerfiles (`forge/containers/Dockerfile.openclaw-sandbox*`)

## Podman Support in OpenClaw

### Gateway (Running OpenClaw Itself)

**Officially supported.** The upstream repo ships:

- `openclaw.podman.env` — Podman-specific environment config
- `scripts/run-openclaw-podman.sh` — Full Podman launcher with rootless support, `--userns=keep-id`, token generation, and Quadlet/systemd integration
- `setup-podman.sh` — One-time host setup (referenced but not in local checkout)

The launcher already handles:
- Rootless Podman with `--userns=keep-id` (maps host UID into container)
- `--init` for proper signal handling (equivalent to Docker's `init: true`)
- `--replace` for idempotent restarts
- Volume mounts with correct ownership

### Agent Sandbox Containers

**Partially compatible with known limitations.** OpenClaw's sandbox code (`src/agents/sandbox/docker.ts`) shells out to the `docker` CLI. Podman provides a `docker`-compatible CLI via `podman-docker` (alias or symlink).

#### What Works

| Feature | Status | Notes |
|---------|--------|-------|
| Container create/start/exec | Works | Podman's CLI is Docker-compatible |
| `--cap-drop ALL` | Works | Same syntax |
| `--network none` | Works | Default sandbox network isolation |
| `sleep infinity` entrypoint | Works | Standard container behavior |
| `--tmpfs` mounts | Works | Same syntax |
| Volume bind mounts | Works | Same syntax |
| Seccomp profiles | Works | Same JSON format |
| `--userns=keep-id` (rootless) | Works | Podman-specific advantage |
| Image building (Dockerfiles) | Works | `podman build` is compatible |

#### Known Limitations

| Feature | Issue | Impact |
|---------|-------|--------|
| Bridge networking | Rootless Podman uses `pasta` (not `veth`). Bridge mode requires reconfiguring every service to bind to a bridge-routable IP. Non-trivial due to `/dev/net/tun` limitations | Agents needing network access (package installs, web research) require workarounds |
| `docker.sock` blocking | OpenClaw blocks `/var/run/docker.sock` bind mounts for security. Podman has no daemon socket by default | Implicitly safer, but code may check for socket existence |
| Container inspection | Some `docker inspect` JSON fields differ between Docker and Podman | May cause issues in `dockerContainerState()` parsing |
| Host networking | `--network=host` behaves differently in rootless Podman (limited by user namespace) | Unlikely issue — sandbox defaults to `none` |

## Security Comparison

| Aspect | Docker | Podman |
|--------|--------|--------|
| Daemon | Root daemon required | Daemonless, no root |
| Container escape impact | Attacker gets root (or Docker group = root-equivalent) | Attacker gets unprivileged user |
| User namespaces | Opt-in (`userns-remap`) | Default (rootless) |
| Socket exposure | `/var/run/docker.sock` is an attack vector | No socket |
| Logging | Centralized via daemon | Per-user via journald |
| Attack surface | Larger (daemon + API) | Smaller |

**Verdict:** Podman is inherently more secure for sandboxing AI agents. A container escape in Podman-rootless lands as an unprivileged user who can read files owned by that user but cannot compromise the host.

## Networking Deep Dive

The critical difference is **bridge networking**. In Docker:

```
Container → veth → docker0 bridge → host network → internet
```

In rootless Podman:

```
Container → pasta/slirp4netns → user-space NAT → host network → internet
```

Pasta (Podman's default network backend since v5.0) provides better performance than slirp4netns but has limitations:
- Port forwarding works but requires explicit `-p` mapping
- Container-to-container communication needs a Podman network (`podman network create`)
- No transparent bridge — containers cannot reach each other by IP without setup

For sandbox containers with `network: "none"` (the default), **this is irrelevant** — no networking is used.

For sandbox containers needing network (`network: "bridge"` for `setupCommand` or agent web access):
- Use `podman network create openclaw-sandbox-net` to create a dedicated network
- Map sandbox config `network: "openclaw-sandbox-net"` instead of `"bridge"`
- Or use `--network=slirp4netns:allow_host_loopback=true` for host-only access

## Existing Podman Infrastructure in b4arena

| File | Purpose | Status |
|------|---------|--------|
| `resources/openclaw/openclaw.podman.env` | Podman env template | Ready |
| `resources/openclaw/scripts/run-openclaw-podman.sh` | Gateway launcher | Ready, production-quality |
| `forge/containers/Dockerfile.openclaw-sandbox` | Sandbox base image | Compatible (standard Dockerfile) |
| `forge/containers/Dockerfile.openclaw-sandbox-common` | Dev sandbox image | Compatible (standard Dockerfile) |
| `infra/templates/openclaw-*.j2` | Infrastructure templates | Need audit for Docker references |

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Sandbox networking breaks | Medium | High — agents cannot install packages or access web | Pre-test with dedicated Podman network; keep Docker as fallback |
| `docker inspect` output parsing fails | Low | Medium — container state detection wrong | Test `podman-docker` compatibility; patch if needed |
| Image build differences | Very Low | Low — Dockerfiles are standard | Podman's Buildah is fully OCI-compatible |
| Performance regression | Very Low | Low — Podman is comparable to Docker | Benchmark container startup time |
| `setupCommand` fails | Medium | Medium — custom tool installs broken | Test with `network != "none"` and `user: "0:0"` |

## References

- [OpenClaw Sandboxing Docs](https://docs.openclaw.ai/gateway/sandboxing)
- [OpenClaw Docker Setup](https://docs.openclaw.ai/install/docker)
- [Security Hardening Guide](https://aimaker.substack.com/p/openclaw-security-hardening-guide)
- [Simon Willison: Running OpenClaw in Docker](https://til.simonwillison.net/llms/openclaw-docker)
- [OpenClaw Security: Sandboxing AI Agents](https://accuknox.com/blog/openclaw-security-ai-agent-sandboxing-aispm)
- [Sandbox Default Image Issue #10361](https://github.com/openclaw/openclaw/issues/10361)
- `src/agents/sandbox/docker.ts` — Container lifecycle implementation
- `src/agents/sandbox/types.docker.ts` — Docker config type definitions
- `exploration/deployment/sandbox-container-lifecycle.md` — Full lifecycle documentation
