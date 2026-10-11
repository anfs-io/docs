---
status: complete
started_at: "2026-09-15T09:05:00+08:00"
completed_at: "2026-09-15T09:40:00+08:00"
deviations: brew_env is called right after library sourcing instead of inside main(), so commands that bypass main() (package) also get Homebrew on PATH
summary: Added lib/platform.sh (platform, brew location/ownership, owner-only brew update on every platform); stack/zsh sets up brew from one prefix-agnostic block that loads first
---

# Execution Log

## What Was Done

- New `lib/platform.sh`:
  - `platform()` — `macos`, or `debian` from os-release `ID`/`ID_LIKE` (`PPM_OS_RELEASE` overrides the file); same logic as `install.sh`
  - `brew_prefix()` — `/opt/homebrew`, `/usr/local`, `/home/linuxbrew/.linuxbrew` (`PPM_BREW_PREFIXES`)
  - `brew_env()` — `brew shellenv bash` when brew is not already on PATH
  - `brew_owner [prefix]` — owner of `<prefix>/Homebrew` or the prefix; returns 1 without Homebrew
  - `brew_is_owner()`, `brew_require_owner <action>` — the latter queues `Homebrew is owned by <owner>; ask them to run: <action>` via `ppm_fail`
  - `update_brew_if_needed()` moved here from `lib/sources.sh`; skips with a debug message when brew is missing or the user is not the owner
- `ppm`: calls `brew_env` after sourcing libraries (deviation: not in `main()`, so `ppm package` is covered too); header lists `platform.sh`
- `lib/installer.sh`: `install()` calls `update_brew_if_needed` on every platform (was macOS only)
- `stack/zsh`: one block at the top of `aliases.zsh` evals `brew shellenv zsh` from the first existing prefix when `HOMEBREW_PREFIX` is unset, before `$BIN_DIR` is prepended (so `~/.local/bin` stays first). Removed the hardcoded blocks from `macos.zsh` and `linux.zsh`. Because `aliases.zsh` loads first, every other `.zsh` file now sees brew tools (previously `claude.zsh`, `fnox.zsh`, `git.zsh`, … loaded before `linux.zsh`/`macos.zsh`)
- `CLAUDE.md`: `platform.sh` in Lib Structure; new "Homebrew Ownership" section

## Test Results

### macOS (bash 3.2)

- `bash -n` on `ppm` and all libs
- Functions: `platform` (macos; ubuntu → debian), `brew_prefix`, `brew_owner`, `brew_is_owner`, `brew_require_owner` for the owner; `brew_owner` returns 1 without brew; with `id -un` overridden to another user: `brew_is_owner` false, `update_brew_if_needed` skips without running brew or writing the cache, `brew_require_owner` queues the owner message; without brew it reports Homebrew missing — all pass
- `env -i HOME=… PATH=/usr/bin:/bin ppm show stack/git` works (yq found through `brew_env`); `ppm list` output unchanged (61 packages)
- zsh block in `zsh -f` with a bare PATH: `HOMEBREW_PREFIX=/opt/homebrew`, brew on PATH, `path[1]=~/.local/bin`, `path[2]=/opt/homebrew/bin`, loop variable not leaked; a preset `HOMEBREW_PREFIX` is left alone

### Debian 13 container (owner + other from plan 01)

Changed ppm files copied into both users' clones, changed pde files into owner's clone.

- other, bare PATH: `ppm install --debug -s ppm/ppm` logs `Homebrew is owned by owner; skipping brew update`, installs, writes no brew cache; `ppm show stack/zsh` finds yq
- owner, bare PATH: first `ppm install` runs `brew update` once and writes the cache; second run does not update
- owner interactive zsh (`env -i … zsh -ic`): `HOMEBREW_PREFIX=/home/linuxbrew/.linuxbrew`, brew on PATH, `path[1]=/home/owner/.local/bin`

## Context Updates

- Plan 03 can use `brew_is_owner` / `brew_require_owner` for consumer handling and `platform()` for `system` map resolution
- Intel Macs dropped after this plan: `/usr/local` removed from `brew_prefix` (ppm, install.sh) and the stack/zsh block; macOS support means Apple silicon only
