---
---

# Plan 02 — Brew Runtime

## Context — read these files first

- `ppm` — bootstrap and `main()`
- `lib/core.sh` — `os()`, `arch()` (package-facing API; unchanged)
- `lib/sources.sh` — `update_brew_if_needed()` (moves in this plan)
- `lib/installer.sh` — `install()` calls `update_brew_if_needed` on macOS only
- `install.sh` — `platform()` and `brew_prefix()` from plan 01 (must stay identical to the lib versions)
- `stack/packages/zsh/home/.config/zsh/macos.zsh` and `linux.zsh` — each hardcodes one brew path for `brew shellenv`
- `chorus/units/cross-platform.md` — owner model

## Overview

ppm assumes brew is already on PATH (from the caller's zsh setup) and only refreshes it on macOS. Make ppm find and use brew itself on every platform, know whether the current user owns the prefix, and only do owner-only work (updates, installs) as the owner. Add a new domain lib, `lib/platform.sh`, for platform, distro and package-manager concerns that plan 03 builds on.

## Implementation

### 1. `lib/platform.sh`

- `platform()` — `macos`, or the supported match from `/etc/os-release` `ID`/`ID_LIKE` (`debian` now). Same logic as the installer's copy.
- `brew_prefix()` — first existing of `/opt/homebrew`, `/home/linuxbrew/.linuxbrew`.
- `brew_env()` — if `brew` is not on PATH and a prefix exists, `eval "$("$prefix/bin/brew" shellenv bash)"`.
- `brew_owner()` — owner of the prefix directory (`stat -f %Su` on macOS, `stat -c %U` on Linux).
- `brew_is_owner()` — `[[ "$(brew_owner)" == "$(id -un)" ]]`.

Move `update_brew_if_needed()` here from `lib/sources.sh`.

### 2. Use it from ppm

- `main()` calls `brew_env` after sourcing config, so ppm works from non-login shells, cron and `sudo -iu` sessions.
- `install()` calls `update_brew_if_needed` on every platform, not just macOS.
- `update_brew_if_needed` returns immediately when there is no brew or the user is not the owner (`debug` the reason). The cache file stays per user in `$PPM_CACHE_HOME`.

### 3. Shell integration (stack/zsh)

Replace the per-OS hardcoded blocks in `macos.zsh` and `linux.zsh` with one prefix-agnostic block (e.g. in `aliases.zsh`, before anything that needs brew tools):

```zsh
if [[ -z $HOMEBREW_PREFIX ]]; then
  for p in /opt/homebrew /home/linuxbrew/.linuxbrew; do
    [[ -x $p/bin/brew ]] && { eval "$($p/bin/brew shellenv zsh)"; break; }
  done
fi
```

Consumer users get the same PATH as the owner; they never run `brew install`.

### 4. Owner messages

Add `brew_require_owner <action>` for plan 03: prints `brew is owned by <owner>; ask them to run: <action>` and returns 1 when the user is not the owner.

### 5. Docs

`CLAUDE.md`: add `lib/platform.sh` to the Lib Structure, describe the owner model in one paragraph.

## Test Spec

- macOS owner: `env -i HOME=$HOME PATH=/usr/bin:/bin ~/.local/bin/ppm list` works (brew found without shell setup)
- Debian 13 owner: same, with `/home/linuxbrew/.linuxbrew`
- Debian 13 owner: `ppm install <pkg>` runs `brew update` at most once per `HOMEBREW_UPDATE_CACHE_DURATION`
- Second user (consumer) on both platforms: `ppm install <pkg>` never runs `brew update`; `--debug` shows the skip reason
- New zsh shell for owner and consumer: `brew` on PATH, `HOMEBREW_PREFIX` set, on both platforms

## Verification

- [ ] `lib/platform.sh` exists with `platform`, `brew_prefix`, `brew_env`, `brew_owner`, `brew_is_owner`, `update_brew_if_needed`
- [ ] No hardcoded brew prefix in ppm outside `brew_prefix()`, or in stack/zsh outside the single shellenv block
- [ ] `update_brew_if_needed` is owner-only and runs on Linux
- [ ] Installer and lib copies of `platform()`/`brew_prefix()` match
