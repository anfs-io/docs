---
status: complete
started_at: "2026-09-17T09:00:00+08:00"
completed_at: "2026-09-17T10:45:00+08:00"
deviations: >-
  gam and ansible keep their `mise install` call because a fix-up step runs immediately after it;
  the zsh login-shell hook runs chsh with stdin closed (a password prompt would block a
  non-interactive install) and compares resolved shell paths
summary: all packages in pde, pdt, ai and user migrated to declared dependencies; install_dep removed from ppm; verified on macOS and by an end-to-end container run on Debian 13
---

# Execution Log

## What Was Done

### pde
- Deleted `install.sh` where it only declared software: fnox, ghostty, git, mise, op
- `git`: `brew: [gh]`, dropping the apt keyring block; `mise`: `brew: [mise]`, dropping the apt keyring block and cosign/gpg; `nvim`: `brew: [neovim, ripgrep]`, dropping the AppImage/FUSE path (hook now only runs Lazy sync); `tmux`: `brew: [tmux, entr]`; `network`-adjacent tools unchanged
- `zsh`: `brew: [bat, fzf, htop, tree]` + `system: {linux: [zsh]}` (the login shell must be a path in `/etc/shells`); hook keeps oh-my-zsh clones and sets the login shell without sudo
- `ruby`: `brew: {macos: […]}` + `system: {debian: […]}`; hook keeps the default-gems file, which ppm's `mise install` then uses
- `ssh`: `system: {debian: [sshfs]}`; `tailscale`: `cask: {macos: [tailscale-app]}` with the Linux vendor installer kept in the hook (it adds a repo and a systemd service)
- `ghostty`, `hammerspoon`: `platforms: [macos]` + cask; hammerspoon's Spoons downloads moved to `post_install`
- `op`: `cask: {macos: [1password, 1password-cli]}`; the mise toml moved to `linux/.config/mise/conf.d/op.toml` so macOS uses the cask and Linux the per-user mise tool
- `linux.zsh`: the `batcat` alias is now conditional, since bat comes from brew rather than apt

### pdt
- Deleted `install.sh`: elixir, ethereum, network-tools, node, opentofu, packer, pcs, pim, python, varlock
- Declarations: `elixir` (brew.macos openssl + system.debian build deps, `libncurses5-dev` → `libncurses-dev` for Debian 13), `network-tools` (brew), `nfs` (system.debian, hook keeps exports/systemd and macOS autofs), `pcs` (`platforms: [linux]` + system.debian), `phoenix` (system.debian inotify-tools), `pim` (brew.macos + system.debian), `podman` (brew.macos + system.linux, hooks keep the socket and machine setup), `solana` (system.debian build deps for the disabled hooks), `utm` (`platforms: [macos]` + cask, hook keeps the utmctl link), `varlock` (`brew: {macos: [dmno-dev/tap/varlock]}`, new package.yml)
- `rust`: hook keeps `rustup component add`, drops `mise install rust`
- Removed the retired `os:` key from network-tools, nfs and ventoy

### ai
- Deleted `install.sh`: antigravity, herdr, hermes, opencode, pi
- `antigravity`: `platforms: [macos]` + casks; `herdr`/`hermes`: `brew: {macos: […]}` (Homebrew only ships macOS binaries for them); `claude`, `opencode`, `pi`: mise as decided, hooks lose their `mise install` lines; `omlx` gained a package.yml (`platforms: [macos]`) and keeps its hook, which needs a custom tap and build option

### user
- `ssh`: `brew: {macos: [lazyssh]}`, `install.sh` deleted; `host` keeps its Brewfile hook; the chorus packages are unchanged

### ppm
- `install_dep` removed from `lib/core.sh`; `CLAUDE.md` hook API and lib description updated
- `git/hooks/lib/package-meta.sh`: `meta_bootstrap` no longer writes the retired `os:` key, and the now-unused `meta_detect_os` is gone

## Test Results

### macOS

- Syntax clean for `ppm`, all libs, the git hook library and every remaining package hook
- No live `install_dep` callers anywhere (three commented-out mentions remain), no apt or keyring code in hooks, no `^os:` keys left
- `ppm deps git zsh nvim` order unchanged
- `ppm show` resolves per platform for zsh, ruby, op, ghostty, git, tmux, nvim, ssh, tailscale, mise
- Declared-vs-installed report: nothing declared is missing except software this Mac never had (entr, iperf3, nmap, hermes-agent, lazyssh, and the 1password/utm/antigravity apps). None of those apps exist outside brew here, so installing those packages later will not collide with hand-installed copies

### Debian 13 container (reset from the brew snapshot, host working tree mounted)

| Check | Result |
|---|---|
| `install -r stack/zsh stack/git stack/tmux stack/podman` | exit 0, one sudo prompt: system batch (podman, podman-compose) then brew batch (bat, fzf, htop, tree, gh, tmux, entr) |
| What landed | all seven formulas; apt zsh, podman, podman-compose |
| Trackers | `stack/zsh` records bat/fzf/htop/tree under `installed_deps.brew`; `stack/podman` records none (system packages are never recorded) |
| `install stack/claude` | `mise install claude`, no prompts; `mise which claude` resolves |
| Rerun `install stack/zsh` | nothing installed, no prompts, no hang |
| `remove stack/tmux` | uninstalls tmux and entr (brew also drops their orphaned dependencies) |
| Login shell | already zsh: no message, no `chsh` attempt |

## Findings

- **`chsh` hangs a non-interactive install.** The first container run stalled for an hour: `chsh` asks for the user's password through PAM and waits forever when nothing can answer, and later ppm runs queued behind it. The hook now runs `chsh </dev/null`, so it fails immediately and prints the command to run by hand. Any hook that can prompt has this problem; hooks should close stdin or check for a TTY.
- `/bin/zsh` and `/usr/bin/zsh` are the same file on Debian, so the shell check resolves both paths before comparing.
- Homebrew 7 removes orphaned dependencies along with an uninstalled formula (libevent, utf8proc with tmux), which is brew's behavior, not ppm's.

## Context Updates

- The cross-platform unit's code work is complete; plan 01's VM cases (real sudo prompt, macOS second user, `--repo`, x86_64) still need real machines
- Next unit: distros (Fedora), which adds `system.fedora` entries to the packages migrated here
