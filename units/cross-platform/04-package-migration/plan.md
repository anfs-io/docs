---
---

# Plan 04 — Package Migration

## Context — read these files first

- `chorus/units/cross-platform/03-declarative-deps/plan.md` — the `package.yml` schema this plan migrates to
- Every `install.sh` and `package.yml` in the table below (repos under `~/.local/share/ppm/`)
- `lib/core.sh` — `install_dep()` (removed at the end)
- `git/hooks/lib/package-meta.sh` — `meta_bootstrap()` writes the retired `os:` key
- `README.md`, `CLAUDE.md` — document `install_dep` and the old model

## Overview

Move every package to declared dependencies, drop `install_dep` calls and per-package `mise install` calls, and remove `install_dep` from ppm. Hooks keep imperative work only (services, generated config, vendor setup with no brew formula).

## Decisions

- **AI agents (claude, opencode, pi) stay on mise.** Homebrew could provide claude-code and opencode on both platforms, but only the Homebrew owner can upgrade them, and agents release constantly; mise installs per user so everyone updates their own. `pi` has no Homebrew package, so brew would split the three across two installers. mise also offers `minimum_release_age` to hold back fresh releases of tools with this much access. Migration is only deleting the `mise install` lines from their hooks.
- **`op`: cask on macOS, mise on Linux.** macOS pairs `1password-cli` with the `1password` app cask (both already casks, and `op` releases are infrequent, so the owner bottleneck costs little). Linux hosts use service accounts (`OP_SERVICE_ACCOUNT_TOKEN` via `rssh`), which never touch the desktop app, so the per-user mise install is right there. Desktop-app integration on Linux would need `op` in `/usr/local/bin`, group `onepassword-cli`, setgid — only 1Password's apt/dnf packages do that, and a setgid binary under a user-writable mise path would be unsafe. Add the vendor repo in a Linux hook only when a Linux desktop actually needs it.
- **Casks are not macOS-only.** Homebrew installs casks on Linux (verified: `claude-code`, `1password-cli`). GUI apps that only exist on macOS are declared as `cask: {macos: [...]}`.

## Migration Rules

1. Same `install_dep` names in `install_macos` and `install_linux` → `brew: [...]`; delete both functions if nothing else is in them.
2. `install_dep --cask` → `cask: [...]`; GUI apps that exist only on macOS become `cask: {macos: [...]}` plus `platforms: [macos]` when the whole package is macOS-only.
3. macOS-only tools (Apple frameworks, macOS apps) → `platforms: [macos]`.
4. Linux software that integrates with the system (services, kernel/FUSE, KVM, disk tools, login shell) → `system: { debian: [...] }` (or `linux:` when names are distro-neutral). macOS equivalents from brew go in `brew: { macos: [...] }`.
5. Build dependencies for mise-compiled runtimes → `brew.macos` + `system.debian` (Linux build headers come from the distro).
6. apt keyrings and curl-to-shell installers → `brew` when a formula exists; otherwise keep in the hook and add a comment explaining why.
7. `mise install <tool>` / `source <(mise activate bash)` in hooks → delete; the stowed `conf.d/<tool>.toml` plus ppm's single `mise install` covers it. Hooks that run the tool right after install call `mise exec <tool> -- …`.
8. Remove the retired `os: [linux, macos]` list; use `platforms` only when the package is not supported everywhere.
9. Check each Linux `brew` formula exists and has a bottle on Linux (`brew info --json=v2 <name>`); if not, move it to `system` or mark the package `platforms: [macos]`.

## Package Table

Tags from the survey: `mac`/`linux` = has `install_<os>`, `dep` = calls `install_dep`, `cask`, `mise` = `mise install` in hook, `apt`/`sudo`/`curlsh`/`brew` = raw usage.

### ai

| Package | Today | Target |
|---|---|---|
| antigravity | mac cask: antigravity, antigravity-cli | `cask: [antigravity, antigravity-cli]`, `platforms: [macos]` unless a Linux install exists |
| claude | mise | drop `mise install`; stays on mise (see Decisions) |
| herdr | mac dep: herdr | `brew: [herdr]`; check Linux formula (rule 9) |
| hermes | mac dep: hermes-agent | `brew: [hermes-agent]`; check Linux formula |
| omlx | mac dep + raw brew: omlx | `brew: [omlx]`, `platforms: [macos]` (MLX is Apple silicon only); move raw brew calls out of the hook |
| opencode | mise | drop `mise install`; stays on mise |
| pi | mise | drop `mise install`; stays on mise (no Homebrew package) |

### pde

| Package | Today | Target |
|---|---|---|
| fnox | mise | drop `mise install` |
| gam | mise | drop `mise install` |
| ghostty | mac cask: ghostty | `cask: [ghostty]`, `platforms: [macos]` (Debian has no official package) |
| git | linux apt keyring for gh | `brew: [gh]`; delete the keyring code |
| hammerspoon | mac dep: hammerspoon | `cask: [hammerspoon]`, `platforms: [macos]` |
| mise | apt keyring + brew: cosign gpg mise | `brew: [mise]` (installer already provides it, plan 01); drop cosign/gpg unless a hook still verifies signatures |
| nvim | linux: libfuse2 fuse3 build-essential ripgrep gcc; mac: neovim ripgrep | `brew: [neovim, ripgrep]`; delete the Linux AppImage/FUSE path |
| op | mac cask 1password + mise op | `cask: {macos: [1password, 1password-cli]}`; on Linux keep mise: move `home/.config/mise/conf.d/op.toml` to `linux/.config/mise/conf.d/op.toml` (ppm stows `$PPM_GROUP_ID`, and `_mise_install_declared` reads that dir too) so macOS installs the cask and Linux the mise tool; drop `mise install op` from the hook |
| ruby | linux build deps; mac libyaml openssl readline pkgconf; mise | `brew.macos: [libyaml, openssl, readline, pkgconf]`, `system.debian: [build-essential, zlib1g-dev, libssl-dev, libreadline-dev, libyaml-dev]`; drop `mise install` |
| ssh | linux dep sshfs; mac macfuse (commented) | `system.debian: [sshfs]` |
| tailscale | linux curlsh (+sudo); mac cask tailscale-app | `cask.macos: [tailscale-app]`; Linux keeps the vendor installer in `install_linux` (daemon + apt repo), documented exception |
| tmux | linux: entr tmux; mac: tmux | `brew: [tmux, entr]` |
| zsh | linux: bat fzf htop tree zsh (+sudo); mac: bat fzf htop tree | `brew: [bat, fzf, htop, tree]`, `system.linux: [zsh]`; keep a login-shell hook, rewritten: skip when the login shell is already zsh; otherwise use `chsh -s "$(command -v zsh)"` (the user's own password, no sudo) and fall back to a `user_message` with the command to run. Today it runs `sudo usermod` on every install (`usermod: no changes` on rerun), which prompts every time with a password sudo and can never succeed for a user without sudo (found in plan 01 container testing, where `other` got a sudo prompt from `ppm install zsh`). Blocks consumer installs until migrated |

### pdt

| Package | Today | Target |
|---|---|---|
| ansible | mise | drop `mise install` |
| elixir | linux build deps; mac openssl; mise | `brew.macos: [openssl]`, `system.debian: [build-essential, autoconf, m4, libncurses-dev, libssl-dev]` (check `libncurses5-dev` still exists on Debian 13); drop `mise install` |
| ethereum | mise | drop `mise install` |
| network-tools | both: iperf3 nmap; stale `os:` | `brew: [iperf3, nmap]`; remove `os:` |
| nfs | linux: nfs-kernel-server (+sudo); stale `os:` | `system.debian: [nfs-kernel-server]`; remove `os:`; macOS uses the built-in nfsd |
| node | mise | drop `mise install` |
| opentofu | mise | drop `mise install` |
| packer | mise | drop `mise install` |
| pcs | linux: dnsmasq bridge-utils iproute2 nmap iperf3 qemu-system-* | `system.debian: [dnsmasq, bridge-utils, iproute2, qemu-system-arm, qemu-system-x86, qemu-system-data]`, `depends: [network-tools]` for nmap/iperf3, `platforms: [linux]` |
| phoenix | linux: inotify-tools | `system.debian: [inotify-tools]` |
| pim | linux: xz-utils fdisk parted dosfstools expect exfat-fuse qemu-system; mac: qemu socat libarchive | `brew.macos: [qemu, socat, libarchive]`, `system.debian: [xz-utils, fdisk, parted, dosfstools, expect, exfat-fuse, qemu-system]`; note `pim.zsh` hardcodes `/opt/homebrew/share/qemu/…` — use `$HOMEBREW_PREFIX` |
| podman | both: podman podman-compose (+sudo) | `brew.macos: [podman, podman-compose]`, `system.linux: [podman, podman-compose]`; keep the systemd socket and podman machine hooks |
| python | mise | drop `mise install` |
| rust | mise | drop `mise install` |
| solana | disabled `x`-prefixed hooks with linux build deps, curlsh, surfpool | leave hooks disabled; move build deps to `system.debian` so re-enabling needs no `install_dep` |
| utm | mac dep: utm | `cask: [utm]`, `platforms: [macos]` |
| varlock | mac dep: dmno-dev/tap/varlock | `brew: [dmno-dev/tap/varlock]`; check Linux support of the tap |
| ventoy | linux download hook; stale `os:` | `platforms: [linux]`; remove `os:`; keep download hook |

### user

| Package | Today | Target |
|---|---|---|
| host | mac sudo + raw brew | inspect; move formulas to `brew`, keep host-specific sudo steps in the hook with a comment |
| ssh | mac raw brew | inspect; move formulas to `brew` |

## Implementation

### 1. Migrate repo by repo

Order: pde (base tools most packages depend on), pdt, ai, user. For each package apply the rules, then:
- `ppm show <pkg>` shows the expected resolved deps on macOS and Debian 13
- `ppm install -r <pkg>` in the VMs succeeds with no `install_dep` deprecation notes in `--debug`

One commit per repo.

### 2. Stop writing the retired key

In `git/hooks/lib/package-meta.sh`, remove `os:` from `meta_bootstrap()` (and `meta_detect_os()` if nothing else uses it). No other hook changes (hooks refactor is out of scope).

### 3. Remove `install_dep`

After `grep -rn install_dep ~/.local/share/ppm/*/packages` returns nothing: delete `install_dep` from `lib/core.sh`, update `CLAUDE.md` (hook API list) and `README.md`.

## Test Spec

In macOS and Debian 13 VMs, as owner and as a second user:

- `ppm install zsh` on a fresh post-installer machine (default bootstrap package)
- Install each repo's packages (`ppm install -f pde/`, `pdt/`, `ai/`), noting unsupported-platform skips
- Remove and reinstall a sample with shared brew deps (e.g. network-tools + pcs) and confirm removal behavior from plan 03

## Verification

- [ ] `grep -rn 'install_dep' */packages` is empty across all repos
- [ ] No `mise install` or `mise activate bash` in any `install.sh`
- [ ] No `^os:` key in any `package.yml`
- [ ] Every remaining apt/sudo/curl-to-shell use in hooks has a comment explaining why it is not declared
- [ ] `install_dep` removed from `lib/core.sh`; docs updated
- [ ] VM results per repo recorded in `log.md`
