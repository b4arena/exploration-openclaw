# Node.js Isolation Without Containers on Fedora

Research into modern approaches (2025-2026) for installing a Node.js CLI tool via npm in an isolated, cleanly removable way on Fedora Linux — without Docker, Podman, or VMs.

---

## 1. Comparison Table

| Approach | Isolation Level | Cleanup Method | Complexity | Gotchas |
|---|---|---|---|---|
| **npm `--prefix`** | Directory-scoped (`/opt/myapp/`) | `rm -rf /opt/myapp` | Low | No shims; must reference `bin/` directly in systemd. `node_modules/.bin/` for local, `{prefix}/bin/` for global-style. Needs system Node.js installed separately. |
| **Volta** | User-local (`~/.volta/`) | `rm -rf ~/.volta`, remove PATH entries | Low-Medium | Designed for developer UX (project pinning). Global packages get shims in `~/.volta/bin/`. Single-binary install. Not widely used on servers. |
| **fnm** | User-local (`~/.local/share/fnm/`) | `rm -rf` the data dir, remove shell hook | Low | Rust binary, fast. No global-package shim magic like Volta — just switches Node versions. Global npm packages go into the active Node prefix. |
| **mise (ex-rtx)** | User-local (`~/.local/share/mise/`) | `rm -rf` the data dir, remove shell hook | Low-Medium | Successor to asdf, written in Rust. Supports Node + many other tools. Uses `mise.toml` for pinning. Good for multi-tool servers (Node + Python etc.). |
| **Homebrew/Linuxbrew** | Prefix-scoped (`/home/linuxbrew/.linuxbrew/`) | `rm -rf /home/linuxbrew/.linuxbrew`, remove PATH | Medium | Large footprint (~1 GB+). Installs its own GCC toolchain on Linux. `npm install -g` within Homebrew Node stays in the Homebrew prefix. Overkill for a single app. |
| **Bun** | User-local (`~/.bun/`) | `rm -rf ~/.bun`, remove PATH entries | Low | Can install and run most npm packages. Global installs go to `~/.bun/bin/`. Not a drop-in Node.js replacement for all packages (native addons, some Node APIs still missing). `bunfig.toml` bug with `~` expansion (Dec 2025). |
| **Nix (single-package)** | Store-scoped (`/nix/store/`) + profile symlinks | `nix profile remove`, then `nix-collect-garbage` | Medium-High | Best isolation (immutable store). Works on Fedora — `/nix` path recently approved in Fedora. Learning curve for Nix expressions. `nix profile install nixpkgs#nodejs` gives you Node; npm global packages need extra handling. |

---

## 2. Detailed Analysis

### 2.1 npm `--prefix /opt/myapp`

The simplest approach. Requires a system-installed Node.js (e.g., `dnf install nodejs`).

```bash
# Install the CLI tool into a dedicated prefix
npm install --prefix /opt/myapp -g <package-name>
# Creates:
#   /opt/myapp/lib/node_modules/<package-name>/
#   /opt/myapp/bin/<executable>

# systemd unit
ExecStart=/opt/myapp/bin/<executable>
Environment=PATH=/opt/myapp/bin:/usr/bin

# Cleanup
rm -rf /opt/myapp
```

**Pros:** Zero additional tooling. Works with stock Fedora Node.js. Trivial systemd integration.
**Cons:** System Node.js must be installed separately and is shared. If you need a specific Node version, this does not help. The `--prefix` flag is poorly documented and has had bugs (e.g., npm/cli#4467 — `NPM_CONFIG_PREFIX` sometimes ignored).

**Verdict:** Best for "I just need one npm tool deployed and Fedora's Node.js version is fine."

### 2.2 Volta

Volta manages Node.js versions and global packages in `~/.volta/` (configurable via `VOLTA_HOME`).

```bash
# Install
curl https://get.volta.sh | bash
volta install node@22
volta install <package-name>

# Global packages get shims in ~/.volta/bin/
# systemd needs the full path:
ExecStart=/home/deploy/.volta/bin/<executable>

# Cleanup
rm -rf ~/.volta
# Remove from ~/.bashrc: export VOLTA_HOME / PATH additions
```

**Pros:** Single Rust binary. Automatic per-project version pinning via `package.json`. Global packages are shimmed and version-locked.
**Cons:** The shim layer adds indirection — if Volta's shim resolution fails, debugging on a headless server is annoying. Designed primarily for developer workstations. Does not provide Node.js to non-interactive shells without explicit PATH setup.

**Verdict:** Good if you want version pinning and plan to run multiple Node.js projects on the same server.

### 2.3 fnm (Fast Node Manager)

Minimal Rust-based Node version manager. Installs Node versions into `~/.local/share/fnm/node-versions/`.

```bash
# Install
curl -fsSL https://fnm.vercel.app/install | bash
fnm install 22
fnm use 22
npm install -g <package-name>

# Global packages live under the active Node version's prefix:
# ~/.local/share/fnm/node-versions/v22.x.x/installation/bin/

# Cleanup
rm -rf ~/.local/share/fnm
```

**Pros:** Extremely fast. Tiny footprint. Simple mental model — it just manages Node binaries.
**Cons:** No global-package shimming (unlike Volta). The global npm packages are tied to a specific Node version directory. Shell hook needed for version switching (`eval "$(fnm env)"`).

**Verdict:** Lightest-weight version manager. Good for servers where you need exactly one Node version and want clean removal.

### 2.4 mise (formerly rtx)

Universal tool version manager (Node, Python, Ruby, Go, etc.). Written in Rust. Spiritual successor to asdf but faster.

```bash
# Install
curl https://mise.run | sh
mise install node@22
mise use --global node@22
npm install -g <package-name>

# Node installs to ~/.local/share/mise/installs/node/22.x.x/
# Global npm packages go into that prefix's lib/node_modules/

# Cleanup
rm -rf ~/.local/share/mise
```

**Pros:** One tool for all runtimes. Active development, large community. Supports `.tool-versions` (asdf compat), `.nvmrc`, `mise.toml`. Shell hook is fast (Rust).
**Cons:** More features than needed if you only want Node. Configuration file proliferation.

**Verdict:** Best choice if the server also needs Python or other runtimes managed the same way.

### 2.5 Homebrew on Linux (Linuxbrew)

```bash
# Install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
brew install node@22
npm install -g <package-name>

# Global packages go to /home/linuxbrew/.linuxbrew/lib/node_modules/
# Binaries in /home/linuxbrew/.linuxbrew/bin/

# Cleanup
rm -rf /home/linuxbrew/.linuxbrew
# Remove shellenv from ~/.bashrc
```

**Pros:** Familiar to macOS users. Everything stays under one prefix. Large package ecosystem beyond Node.
**Cons:** Installs its own GCC, glibc, etc. — easily 1-2 GB for just Node. Slow initial install. Redundant with Fedora's own package manager for most things. `npm install -g` may need `sudo` workarounds.

**Verdict:** Overkill for isolating a single Node.js app. Only consider if you already use Homebrew on the server for other reasons.

### 2.6 Bun

Bun is a fast JavaScript runtime + package manager. It can install and run many npm packages directly.

```bash
# Install
curl -fsSL https://bun.sh/install | bash
# Installs to ~/.bun/bin/bun

bun install -g <package-name>
# Global packages go to ~/.bun/install/global/
# Binaries linked in ~/.bun/bin/

# Cleanup
rm -rf ~/.bun
```

**Pros:** Extremely fast installs (25x npm in benchmarks). Single binary. Can often replace Node.js entirely for CLI tools.
**Cons:** Not 100% Node.js compatible — native addons (`node-gyp`), some `node:*` APIs, and edge cases may break. The package you want to run must actually work under Bun's runtime. As of early 2026, production-ready for many use cases but not all. Bug with `~` path expansion in `bunfig.toml` (Dec 2025, issue #25766).

**Verdict:** If the specific CLI tool runs under Bun (test it first), this is the simplest and fastest option. If it needs full Node.js compatibility, skip this.

### 2.7 Nix (Single-Package Install)

Nix can be installed on any Linux distro. Fedora has recently approved the `/nix` path, making integration smoother.

```bash
# Install Nix (multi-user, recommended)
sh <(curl -L https://nixos.org/nix/install) --daemon

# Install Node.js into your profile
nix profile install nixpkgs#nodejs_22

# For an npm package, you'd need to either:
# a) Use node2nix to create a Nix expression
# b) Or use the Nix-installed Node and npm install --prefix

# Cleanup of a single package
nix profile list          # find the index
nix profile remove <idx>
nix-collect-garbage        # reclaim store space

# Full Nix removal
sudo systemctl stop nix-daemon
sudo rm -rf /nix /etc/nix ~/.nix-profile ~/.nix-defexpr
```

**Pros:** Best isolation — packages are immutable in `/nix/store/`, never conflict with system packages. Atomic installs and rollbacks. Reproducible. Over 100k packages available. Fedora-friendly now.
**Cons:** Steep learning curve. npm global packages don't work naturally inside Nix (the store is read-only). You'd need `node2nix` or a Nix flake to package an npm tool properly. The `/nix/store` can grow large without regular garbage collection. Multi-user install requires a daemon.

**Verdict:** Best isolation guarantees, but highest complexity. Worth it if you already use Nix or value reproducibility highly.

---

## 3. Recommendation Matrix

| Scenario | Recommended Approach |
|---|---|
| Single npm CLI tool, Fedora's Node version is fine | **npm `--prefix /opt/myapp`** |
| Need specific Node version, single tool, minimal footprint | **fnm** or **Volta** |
| Server runs multiple runtimes (Node + Python + ...) | **mise** |
| The CLI tool runs under Bun (verified) | **Bun** |
| Maximum isolation / reproducibility is the priority | **Nix** |
| Already using Homebrew on the server | **Homebrew** (but don't adopt it just for this) |

### For the OpenClaw use case specifically

Based on `exploration/deployment/iac-setup.md`, the current setup uses system Node.js 22 with a systemd service. The two most practical upgrade paths for better isolation are:

1. **npm `--prefix`** — zero new dependencies, keeps system Node, scopes the app cleanly.
2. **fnm** — if a specific Node.js version must be pinned independently of Fedora's package lifecycle. Ansible can install fnm and pin the version declaratively.

---

## Sources

- [Bun Documentation — Installation](https://bun.com/docs/installation)
- [Bun Package Manager](https://bun.sh/package-manager)
- [Bun as Package Manager (Jan 2026)](https://oneuptime.com/blog/post/2026-01-31-bun-package-manager/view)
- [Bun vs Node.js in 2026](https://pas7.com.ua/blog/en/bun-ready-bun-vs-node-2026)
- [Homebrew — Node for Formula Authors](https://docs.brew.sh/Node-for-Formula-Authors)
- [Linuxbrew + Node.js setup](https://gist.github.com/gerarldlee/ff5790a796b48c14a16bbd49757bb461)
- [npm `--prefix` docs](https://docs.npmjs.com/cli/v7/commands/npm-prefix/)
- [npm `--prefix` issue #11007](https://github.com/npm/npm/issues/11007)
- [npm `NPM_CONFIG_PREFIX` bug #4467](https://github.com/npm/cli/issues/4467)
- [Volta — Understanding Volta](https://docs.volta.sh/guide/understanding)
- [Volta — Installers](https://docs.volta.sh/advanced/installers)
- [fnm on GitHub](https://github.com/Schniz/fnm)
- [mise — Node.js](https://mise.jdx.dev/lang/node.html)
- [mise — Dev Tools](https://mise.jdx.dev/dev-tools/)
- [Getting Started with Mise (Better Stack)](https://betterstack.com/community/guides/scaling-nodejs/mise-explained/)
- [mise Tool Version Management (Jan 2026)](https://oneuptime.com/blog/post/2026-01-25-mise-tool-version-management/view)
- [NVM Alternatives Guide (Better Stack)](https://betterstack.com/community/guides/scaling-nodejs/nvm-alternatives-guide/)
- [Nix `profile install` reference](https://releases.nixos.org/nix/nix-2.19.1/manual/command-ref/new-cli/nix3-profile-install.html)
- [node2nix](https://github.com/svanderburg/node2nix)
- [Fedora `/nix` path approval](https://discourse.nixos.org/t/path-cleared-for-nix-package-manager-on-fedora-with-nix-approved/70897)
- [Fedora Nix Change Proposal](https://fedoraproject.org/wiki/Changes/Nix_package_tool)
- [Uninstalling Nix](https://nixos.org/manual/nix/stable/installation/uninstall)
