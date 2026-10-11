---
status: in_progress
started_at: "2026-09-15T08:10:00+08:00"
completed_at: null
deviations: GitHub host keys are written from GitHub's published known_hosts lines instead of ssh-keyscan plus fingerprint comparison; VM cases outstanding
summary: install.sh reworked for macOS and Debian 13 (prompted sudo, brew owner/consumer, base tools from brew, pinned GitHub host keys); verified on macOS and a Debian 13 arm64 container, VM cases pending
---

# Execution Log

## What Was Done

- Wrapped the whole script in `main()` called on the last line, so a truncated `curl | bash` download executes nothing
- Refuses to run as root (Homebrew refuses too)
- `platform()`: `macos`, or `debian` from `/etc/os-release` `ID` then `ID_LIKE` (Ubuntu, Mint); anything else exits before doing work. `PPM_OS_RELEASE` overrides the file path for testing
- Prerequisites: Debian `build-essential procps curl file git`, checked with `dpkg-query`; sudo requested only when something is missing (`ensure_sudo`: `sudo -n true`, else `sudo -v` with a normal prompt, else exit naming what an admin must install). apt runs with `DEBIAN_FRONTEND=noninteractive`. macOS relies on the Homebrew installer for the Command Line Tools, and exits with `xcode-select --install` guidance if brew exists but the tools don't
- Removed the passwordless-sudo check and the GitHub `yq` download
- Homebrew: `brew_prefix()` checks `/opt/homebrew`, `/usr/local`, `/home/linuxbrew/.linuxbrew`; installs with `NONINTERACTIVE=1` only when none exists; `brew shellenv bash` (was `zsh` inside a bash script)
- Owner/consumer: `brew_owner()` stats `<prefix>/Homebrew` (or the prefix on Apple silicon). Base tools `stow yq mise` (+ `bash` on macOS) are checked via `<prefix>/opt/<formula>` without running brew; the owner installs missing ones, a consumer exits with `Ask <owner> to run: brew install …`
- GitHub host keys: skip if `ssh-keygen -F github.com` (or a plain `github.com` line) exists; otherwise append GitHub's three published keys, embedded in the script. Keys verified three ways: fingerprints recomputed with `ssh-keygen -lf` match GitHub's docs, match a live `ssh-keyscan`, and match `api.github.com/meta`. Guards against joining onto a last line without a newline; `~/.ssh` 700, `known_hosts` 600
- Config: header now says the configs are seeded as regular files and replaced by links when `--repo` installs the repo's ppm package; `install_repo` warns when the repo ships `sources.list` but `~/.config/ppm/sources.list` is not a link, and skips `install -f` when the repo has no ppm package
- Argument parsing: `--repo` without a value and unknown options exit with an error
- README: Quick Start says Debian 13 and non-root; "What Gets Installed" rewritten; new "Multiple Users" section; `--skip-deps` wording

## Test Results

### macOS (this machine), bash 3.2 and 5.3

- `bash -n` passes under `/bin/bash` 3.2 and brew bash 5.3
- Function tests (script sourced without its last line): platform detection (macOS, debian, ubuntu, quoted multi-value `ID_LIKE`, fedora → unsupported, missing os-release), `brew_prefix` (found / none, no errexit trap), `brew_owner`, `base_formulas`, `known_hosts` (fresh + rerun = 3 lines, perms, ed25519 fingerprint, no-trailing-newline file, hashed existing entry), `--repo` without value, unknown option, unsupported platform stops before work — all pass
- Case 9 `--script-only` in a throwaway HOME: clone, `~/.local/bin/ppm` link, `ppm.conf`, `sources.list`, `ppm.local.conf` (`PPM_GROUP_ID=macos`); `~/.ssh` untouched; rerun exits 0

### Debian 13 container (podman, `debian:trixie`, arm64)

Owner has NOPASSWD sudo (no TTY for a prompt); `other` has no sudo. Script: `scratchpad/debian-test/run.sh`.

| Case | Result |
|---|---|
| 5 — no sudo, prerequisites missing | PASS: exits naming `build-essential procps curl file git` for an admin |
| 1 — owner first run | PASS: exit 0; prefix owned by `owner`; `stow yq mise` from arm64 Linux bottles; ppm linked; zsh installed; 3 GitHub lines |
| 2 — owner rerun | PASS: no prerequisite/Homebrew/base-tool installs; known_hosts still 3 lines |
| 3 — second user, base tools present | PASS for the installer (no writes to brew or system; prefix still owned by `owner`); run exits 1 at `ppm install zsh` because `stack/zsh` installs via `install_dep` → `sudo apt` (plan 04) |
| 4 — second user, `yq` missing | PASS: `Homebrew at /home/linuxbrew/.linuxbrew is owned by owner and is missing: yq` / `Ask owner to run: brew install yq` |

## Outstanding (VM)

- Case 1/2 with a password-prompting sudo: exactly one prompt on first run, none on rerun
- Cases 6/7: macOS fresh admin and second standard user
- Case 8: `--repo <url>` leaves `~/.config/ppm/sources.list` as a link into the repo
- x86_64 Debian 13
- Case 3 end-to-end (consumer installs zsh) depends on plan 04's zsh migration

## Findings for Later Plans

- Plan 02: `ppm install` calls `update_brew_if_needed` only on macOS and does not check ownership; a macOS consumer will fail at `brew update` until plan 02
- Plan 04: `stack/zsh` installs its tools with `sudo apt` (blocks consumers) and runs `sudo usermod` to set the login shell on every install (`usermod: no changes` on rerun), which would prompt on every run with a password sudo; it should skip when the shell is already set

## Context Updates

- Container test harness lives in the session scratchpad; move to the repo if it should be reused
