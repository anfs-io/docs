---
objective: It works everywhere — ppm bootstraps macOS and Debian 13 the same way with brew + mise, minimal sudo, and multiple users per machine
status: in_progress
---

Cross-platform tier. Replaces the apt-on-Linux / brew-on-macOS split with one model on both platforms, and replaces the imperative `install_dep` helper with dependencies declared in `package.yml`.

## Decisions

These were settled before planning; plans implement them rather than revisit them.

- **Three tiers of software.**
  - `system` — the distro package manager (apt now; dnf/pacman in the `distros` unit). Only for brew's prerequisites and software that integrates with the system: systemd services, kernel modules, root daemons (podman on Linux, tailscaled, nfs, dnsmasq, qemu/KVM, disk tools), and the login shell.
  - `brew` — command-line tools used system-wide, on macOS and Linux. `cask` is macOS-only (GUI apps, some vendor CLIs).
  - `mise` — language runtimes and tools that may be pinned per project. Packages stow `~/.config/mise/conf.d/<tool>.toml`; ppm runs one `mise install` after stowing. No `mise:` key in `package.yml` — the stowed toml is the declaration.
- **Brew owner model.** Homebrew does not support shared multi-user installations ([Support Tiers](https://docs.brew.sh/Support-Tiers)), and a group-writable prefix on every user's PATH lets any group member replace binaries other users run. One admin user owns the brew prefix and is the only account that installs or upgrades formulas. Other users run the tools (the prefix is world-readable) and never write to it. When a non-owner needs a missing formula, ppm reports what the owner must install.
- **Minimal sudo.** Passwordless sudo is no longer required. sudo is used only to: install brew's prerequisites (once per machine), let the owner create the brew prefix (once), install declared `system` packages that are missing, and set the login shell. Re-running on a configured machine never prompts.
- **Declarative dependencies.** `package.yml` gains:
  - `platforms` — optional list of supported platforms (`macos`, `linux`, `debian`, `fedora`, `arch`); replaces the unused `os: [linux, macos]` list written by the old pre-commit hook
  - `brew`, `cask`, `system` — each either a list (applies on every platform) or a map keyed by platform, resolved as exact platform → `linux` fallback
  - `install_dep` is removed once packages are migrated; `install.sh` hooks remain for imperative work (services, config generation)
- **macOS means Apple silicon.** Intel Macs (Homebrew at `/usr/local`) are not supported.
- **Linux baseline is Debian 13.** Homebrew's Tier 1 Linux requires glibc ≥ 2.39; Debian 12 ships 2.36.
- **Testing happens in VMs**: macOS and Debian 13 (PIM's Debian preseed), each with an admin user (brew owner) and a second, non-admin user.

## Out of Scope

- Fedora and Arch support — the `distros` unit
- PIM kickstart/archinstall support
- Upgrading formulas: ppm installs what is missing; upgrades stay with `brew upgrade` / `mise upgrade`
- Git hooks refactor (plan 04 only stops the pre-commit hook from writing the retired `os:` key)

## Plans

1. `01-installer-bootstrap` — `install.sh`: platform detection, prerequisite bootstrap with prompted sudo, brew install or consumer mode, installer bug fixes
2. `02-brew-runtime` — `lib/platform.sh`: ppm finds and uses brew on every platform and knows whether the current user owns it
2a. `02a-container-harness` — `ppm container` (dev package): standard Debian/Fedora test containers and users, testing the working tree by default
3. `03-declarative-deps` — `platforms`/`brew`/`cask`/`system` in `package.yml`, batch install, tracked removal, single `mise install`
4. `04-package-migration` — move every package in ai, pde, pdt and user repos to the declared model; remove `install_dep`

## Completion Criteria

- `curl … install.sh | bash` succeeds on fresh macOS and Debian 13 VMs as an admin user with a password-prompting sudo; a second run prompts for nothing
- A second, non-admin user on the same machine can install ppm and packages without sudo when the required brew formulas and system packages are present, and gets a clear message naming what is missing otherwise
- No shared or group-writable brew prefix is created; the prefix stays owned by one user
- `ppm` locates brew at `/opt/homebrew` or `/home/linuxbrew/.linuxbrew` without relying on the caller's shell setup
- `package.yml` supports `platforms`, `brew`, `cask` and `system` (list or platform map); `ppm install` installs missing dependencies in one batch per manager before running hooks, and refuses packages whose `platforms` exclude the current platform
- `ppm remove` uninstalls brew formulas and casks that ppm installed for that package and no other installed package declares; system packages are never removed automatically
- No package in ai, pde, pdt or user calls `install_dep`, apt, or a curl-to-shell installer for software available from brew, except documented exceptions
- `install_dep` is gone from `lib/core.sh`; `CLAUDE.md` and `README.md` describe the declared model
