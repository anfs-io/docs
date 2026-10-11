---
---

# Plan 02a — Container Harness

## Context — read these files first

- `packages/dev/home/.local/lib/ppm/dev.sh` — the extension pattern (`user()` becomes `ppm user`)
- `ppm` — `_resolve_path()` (defined before extensions are sourced), `main()` dispatch
- `install.sh` — skips cloning when `~/.local/share/ppm/ppm` exists and config seeding when files exist
- `lib/sources.sh` — `collect_repos()`, `update()` skips entries that are not git URLs (`is_git_url`)
- `chorus/units/cross-platform/01-installer-bootstrap/log.md` — what the ad-hoc Debian container test covered and what containers cannot cover

## Overview

Add `ppm container`, a dev-package extension for testing ppm in disposable Linux containers with standard images and users. It tests the **working tree** by default: host repos are mounted read-only and each test user's ppm data dirs link to them, so edits on the Mac are visible in the container immediately without commits. `--pushed` runs the published GitHub installer instead, as a final check of what real users get.

Containers complement VMs; they do not replace them. No systemd services, login sessions (`chsh`), kernel features (NFS, KVM) or other architectures without emulation. Arch is on hold (its official image is amd64-only); this plan covers Debian and Fedora.

## Implementation

### 1. Layout

```
packages/dev/
  home/.local/lib/ppm/container.sh     # stowed; defines container()
  containers/
    debian/Containerfile
    fedora/Containerfile
```

`container.sh` finds `containers/` from its own real path: `_resolve_path "${BASH_SOURCE[0]}"` gives `.../packages/dev/home/.local/lib/ppm`, and the package dir is four levels up.

`container()` checks `command -v podman` first and fails with `ppm container needs podman (ppm install podman)`. No `depends` on `stack/podman`: the ppm repo is the lowest-priority source and cannot depend on higher-priority repos.

### 2. Standard images

Each Containerfile starts from the official image (`debian:trixie`, `fedora:latest`, both multi-arch) and adds only what a fresh machine plus a terminal needs. It does not add Homebrew prerequisites, because the installer does that:

- `sudo`, terminfo (`ncurses-term`), UTF-8 locale (Debian `locales` + `en_US.UTF-8`; Fedora `glibc-langpack-en`)
- `/etc/gitconfig` with `[safe] directory = *` so git accepts the mounted host repos
- Users:
  - `owner` (uid 1000): sudo with password `owner` (group `sudo` on Debian, `wheel` on Fedora), login shell bash
  - `other` (uid 1001): no sudo, login shell bash
- `CMD ["sleep", "infinity"]`

Images are tagged `localhost/ppm-test:<distro>`.

### 3. Commands

| Command | Behavior |
|---|---|
| `build <distro>` | `podman build -t localhost/ppm-test:<distro> containers/<distro>` |
| `start <distro> [--from <snapshot>] [--sources a,b]` | Build the image if missing, then run `ppm-<distro>` detached with hostname `<distro>`. Mounts each host source (all in the host `sources.list`, or `--sources`) read-only at `/src/<alias>`, plus the host `sources.list` at `/src/sources.list`. |
| `shell <distro> [owner\|other]` | `podman exec -it -e TERM=xterm-256color -u <user> -w /home/<user> ppm-<distro> <login shell> -l`; login shell read from `getent passwd` so it follows `chsh` |
| `install <distro> [owner\|other] [--pushed] [installer args…]` | Working tree (default): link the user's data dirs (step 4), then `podman exec -it -u <user> … bash /src/ppm/install.sh <args>`. `--pushed`: `curl -fsSL https://raw.githubusercontent.com/maxcole/ppm/refs/heads/main/install.sh \| bash -s -- <args>` with no links. `-it` gives a TTY so the sudo password prompt can be answered. |
| `snapshot <distro> <name>` | `podman commit ppm-<distro> localhost/ppm-test:<distro>-<name>` |
| `reset <distro> [<snapshot>]` | Remove the container and `start` again from the base image or the snapshot, with the same mounts |
| `stop <distro>` / `rm <distro>` | `podman stop` / `podman rm -f` |
| `list` | containers named `ppm-*` with status, and `localhost/ppm-test` images/snapshots |

Default user is `owner`. Unknown distro → list the directories under `containers/`.

### 4. Working-tree links

Before running the installer as `<user>` (idempotent, as that user):

```bash
mkdir -p ~/.local/share/ppm ~/.config/ppm
for each /src/<alias> directory:
  ln -sfn /src/<alias> ~/.local/share/ppm/<alias>
write ~/.config/ppm/sources.list with "/src/<alias>  <alias>" in host source order
```

Effects with the current code, no installer or ppm changes:
- `install.sh` finds `~/.local/share/ppm/ppm` and skips cloning; finds `sources.list` and skips seeding it (`ppm.conf` is still seeded)
- `ppm update` and the auto-update timer skip the entries because local paths are not git URLs
- stowed links point through `~/.local/share/ppm/<alias>` into `/src`, so host edits apply on the next `ppm install -r`

Limitation to document in the usage text: commands that write into a repo (`ppm file claim`, `ppm package`) fail on the read-only mount.

### 5. Docs

- `container` usage text lists commands, the working-tree default, `--pushed`, and the container-vs-VM limits
- README "Development": add a short `ppm container` example next to `ppm user`

## Test Spec

On Apple silicon with podman:

- `ppm container build debian` and `build fedora` succeed (arm64)
- `ppm container start debian`; `shell debian` opens a 256-color bash as owner; `shell debian other` as other
- `install debian` as owner: exactly one sudo password prompt (type `owner`); installs Homebrew, base tools and zsh from the working tree; `~/.local/share/ppm/pde` is a link to `/src/pde`
- `install debian` again: no password prompt, nothing installed
- `snapshot debian brew`; `reset debian brew` → owner still has Homebrew, and the reset takes seconds
- Edit a file in a host package (e.g. a zsh alias), `ppm install -r zsh` in the container → change present without commit
- `install debian other` → consumer mode (zsh itself still blocked until plan 04)
- `install debian --pushed` in a fresh container → the GitHub installer runs, and data dirs are clones instead of links
- `start fedora`, `shell fedora` work; `install fedora` exits with the installer's unsupported-platform message (until distros plan 01)
- Without podman on PATH → clear message
- `ppm file claim` inside the container fails cleanly on the read-only mount

## Verification

- [ ] `packages/dev` contains `container.sh` and Containerfiles for debian and fedora
- [ ] Working-tree mode needs no changes to `install.sh` or ppm libs
- [ ] Sudo password prompt is testable interactively
- [ ] Snapshots and reset work
- [ ] Test Spec results recorded in `log.md`
