---
status: complete
started_at: "2026-09-15T09:20:00+08:00"
completed_at: "2026-09-15T10:05:00+08:00"
deviations: Host sources.list is not mounted; source order is stored in a container label (ppm.sources) and reused by install and reset. Debian image adds wget and ca-certificates (present on a real minimal Debian install) so --pushed can fetch the installer.
summary: Added `ppm container` (dev package extension) with Debian and Fedora test images, working-tree mounts, --pushed mode, snapshots and reset; all functional cases pass, remaining sudo prompts traced to stack/zsh (plan 04)
---

# Execution Log

## What Was Done

- `packages/dev/home/.local/lib/ppm/container.sh` — `container()`:
  - `build`, `start [--from SNAPSHOT] [--sources a,b]`, `shell [owner|other]`, `install [owner|other] [--pushed] [args]`, `snapshot`, `reset [snapshot]`, `stop`, `rm`, `list`
  - Finds `containers/` through the stow link (`_resolve_path "${BASH_SOURCE[0]}"`, four levels up)
  - Checks for podman first; no `depends` (ppm is the lowest-priority source)
  - `start` mounts each host source read-only at `/src/<alias>` (resolved real path), in sources.list order, recorded in label `ppm.sources`; requires the ppm source
  - `install` (working tree): links `~/.local/share/ppm/<alias>` → `/src/<alias>` and writes `sources.list` with local paths, then runs `/src/ppm/install.sh`. No installer or ppm changes needed. Refuses a container that already has clones
  - `install --pushed`: downloads the GitHub installer with curl or wget; refuses a container linked to the working tree
  - `podman exec -t` only when ppm has a TTY, so sudo can prompt interactively and scripted use still works
- `packages/dev/containers/debian/Containerfile` (`debian:trixie`) and `fedora/Containerfile` (`fedora:latest`): sudo, `ncurses-term`, UTF-8 locale, `/etc/gitconfig` `safe.directory = *`, users `owner` (uid 1000, sudo/wheel, password `owner`) and `other` (uid 1001, no sudo)
- README "Development": `ppm container` examples and the container-vs-VM note

## Test Results

Apple silicon, podman machine (applehv). Images: debian built in 20s (235 MB), fedora in 14s (219 MB), both arm64.

### Command paths (host)

Usage, distro list resolved through the stow link, unknown subcommand/distro/user, missing snapshot, shell on a stopped container, invalid snapshot name, `list`, podman missing from PATH — all behave as specified.

### Functional (scripted; `expect` answers sudo prompts through a pty)

| Case | Result |
|---|---|
| A — owner first install, working tree | exit 0 in 100s; `~/.local/share/ppm/pde` → `/src/pde`; Homebrew owned by owner; `sources.list` uses `/src/<alias>` paths. 2 sudo prompts: the installer's one, plus one from `stack/zsh` (see Findings) |
| B — owner rerun | exit 0; installer printed no `sudo is needed` and installed nothing; the 1 prompt comes after `Install stack/zsh` (verified with a full transcript) |
| C — snapshot + reset | `snapshot debian brew`, `reset debian brew` in 11s; brew and links present; `ppm.sources` label preserved |
| D — working tree | a file created on the host appears at `/src/ppm` immediately; `/src` is read-only |
| E — other (no TTY, no sudo) | nothing installed into brew/system; prefix still owned by owner; links created; exits 1 at `stack/zsh` `sudo apt` (plan 04) |
| F — file claim | fails (exit 1) and leaves the original link intact |
| G — shell | login shell as owner and as other, `TERM=xterm-256color` |
| H — fedora | image runs; users and uids correct; owner's sudo requires a password; `install fedora` stops with `Unsupported platform` |
| I — `--pushed` in a fresh container | exit 0 in 113s; data dirs are git clones, not links; a later working-tree `install` refuses the container. Prompt count 2, same pattern as A |

Final state: `ppm-debian` running from `localhost/ppm-test:debian-brew`; images `debian`, `fedora`, `debian-brew`.

### Harness issues found and fixed during testing (test script only)

- `expect -c … -- args` does not set `$argv`; moved the pty helper to `tty.exp`
- `log_file` wrote empty transcripts, which also made the first "B: nothing installed" check meaningless; the helper now echoes to stdout and the caller redirects to the transcript, and case B was re-verified from a full transcript
- The shell regex matched the typed command echoed by the terminal; it now excludes `$`

## Findings for Later Plans

- Plan 04: `stack/zsh` prompts for sudo on every install (its `install_dep` → `sudo apt` and `sudo usermod`), including reruns. The installer's own sudo credentials did not carry over to that later prompt in the first install. Plan 03's batched `system` install should be checked for this: one prompt for the whole run is the goal
- The earlier ad-hoc container `ppm-install-test` (plan 01) is still running and can be removed

## Context Updates

- `ppm container` is the default way to test plans 03 and 04 on Debian; `reset debian brew` skips the Homebrew bootstrap
- The test script and `tty.exp` live in the session scratchpad; move them into the repo if they should be kept
