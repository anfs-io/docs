---
---

# Plan 01 — Installer Bootstrap

## Context — read these files first

- `install.sh` — the bootstrap installer being reworked
- `lib/core.sh` — `os()`, `arch()`; the installer carries its own copies because it runs before ppm is cloned
- `lib/sources.sh` — `src add`, `update`
- `README.md` — Quick Start and Advanced Installation sections document installer flags
- `units/cross-platform.md` — decisions (brew owner model, minimal sudo, Debian 13)
- https://docs.brew.sh/Homebrew-on-Linux — prerequisites and prefix

## Overview

`install.sh` today requires passwordless sudo on Linux, installs base tools with apt on Linux and brew on macOS, downloads `yq` from GitHub on Linux, and hardcodes `/opt/homebrew`. Rework it so both platforms follow one flow:

1. Detect platform and distro
2. Install brew's prerequisites with the distro package manager, prompting for sudo only if something is missing
3. Install Homebrew if no prefix exists (this user becomes the owner), or use an existing prefix as a consumer
4. Install ppm's base tools from brew (owner), or verify they are present (consumer)
5. Clone ppm, seed config, add the user's repo, install packages

Fix the installer bugs found in review along the way.

## Implementation

### 1. Platform and distro detection

Replace the installer's `os()` with a `platform()` that prints `macos`, or the first supported match among `ID` then `ID_LIKE` from `/etc/os-release`. Supported now: `debian` (covers Ubuntu via `ID_LIKE`). Anything else exits with a clear "unsupported distro: <ID>" message. Keep the output values aligned with what `lib/core.sh` will expose in plan 02 (`fedora`, `arch` are added in the `distros` unit).

### 2. Prerequisites with prompted sudo

Remove the `NOPASSWD: ALL` check in `setup_deps_linux`.

| Platform | Prerequisites |
|---|---|
| debian | `build-essential procps curl file git` |
| macos | Xcode Command Line Tools (the Homebrew installer handles these) |

- Check each prerequisite first (`dpkg-query -W -f='${Status}'`/`command -v`); build the missing list.
- Only when the list is non-empty: explain what will be installed, run `sudo -v` (normal password prompt), then install.
- If the user cannot sudo and prerequisites are missing, exit naming the packages an admin must install.

### 3. Homebrew: owner or consumer

Add `brew_prefix()` that checks, in order: `/opt/homebrew`, `/home/linuxbrew/.linuxbrew`. Print the prefix or nothing. (Intel Macs are not supported.)

- **No prefix:** install Homebrew with the official installer (`NONINTERACTIVE=1` after an explicit notice; it needs sudo once to create the prefix). This user is now the owner.
- **Prefix owned by this user:** owner mode.
- **Prefix owned by someone else:** consumer mode. Never write to the prefix, never run `brew install`.

Load the environment with `eval "$("$prefix/bin/brew" shellenv bash)"` (the current script is bash; today it passes `zsh`).

### 4. Base tools from brew

Base tools: `stow yq mise`, plus `bash` on macOS (ppm needs a modern bash there). `git` comes from the prerequisites.

- Owner: `brew install` the missing ones in one call.
- Consumer: if any are missing, exit listing them and the owning user: `ask <owner> to run: brew install <missing>`.

Delete the GitHub `yq` download from `setup_deps_linux`.

### 5. GitHub host key

Replace the unconditional `ssh-keyscan github.com >> ~/.ssh/known_hosts` (duplicates on every run, trust-on-first-use):

- Skip when `ssh-keygen -F github.com` already finds an entry.
- Otherwise append GitHub's published keys and verify the ed25519 fingerprint matches GitHub's documented value (check https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints at implementation time) before writing; abort on mismatch.
- Create `~/.ssh` with mode 700.

### 6. Config files

- Fix the header: the configs are seeded as regular files from the template repo, then replaced by links when a `--repo` provides a `ppm` package (`ppm install -f <repo>/ppm`). It never symlinks them directly.
- After `install_repo`, if the repo has `packages/ppm/home/.config/ppm`, verify `~/.config/ppm/sources.list` is a symlink and warn otherwise.
- Keep `PPM_GROUP_ID=$(os)` in `ppm.local.conf` (still `macos`/`linux`; stow subdirs are not per distro).

### 7. Idempotence

Every step checks before acting: prerequisites, brew prefix, base tools, ppm clone, config files, known_hosts. A second run on a configured machine performs no sudo and no network installs beyond `ppm update`.

### 8. Documentation

Update the header comment (what it does, files created, external fetches, sudo usage) and README install sections: no passwordless sudo, owner vs consumer, Debian 13 baseline.

## Test Spec

Run in VMs, snapshot before each case.

| # | Machine | User | Case | Expected |
|---|---|---|---|---|
| 1 | Debian 13 fresh | admin (sudo with password) | `install.sh` | one sudo prompt; brew at `/home/linuxbrew/.linuxbrew` owned by admin; ppm + zsh installed |
| 2 | after 1 | admin | `install.sh` again | no sudo prompt, no brew installs, no duplicate known_hosts entries |
| 3 | after 1 | second user, not in sudo | `install.sh` | consumer mode; no sudo; ppm installed; brew prefix untouched |
| 4 | after 1, remove `yq` | second user | `install.sh` | exits naming `yq` and the owner |
| 5 | Debian 13 fresh | user without sudo | `install.sh` | exits naming prerequisites an admin must install |
| 6 | macOS fresh | admin | `install.sh` | brew at `/opt/homebrew`; ppm + zsh installed |
| 7 | after 6 | second standard user | `install.sh` | consumer mode; no sudo |
| 8 | any | admin | `install.sh --repo <url>` | `~/.config/ppm/sources.list` is a symlink into the repo |
| 9 | any | admin | `install.sh --script-only` | only ppm clone + config |

## Verification

- [ ] No `NOPASSWD` check and no GitHub `yq` download remain in `install.sh`
- [ ] No hardcoded `/opt/homebrew` outside `brew_prefix()`
- [ ] Brew prefix is never chmod/chgrp'd by the installer
- [ ] Cases 1–9 pass; results recorded in `log.md`
- [ ] README and header comment match the behavior
