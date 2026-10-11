---
---

# Plan 01 — Fedora

## Context — read these files first

- `install.sh` — `platform()`, prerequisite bootstrap (cross-platform plan 01)
- `lib/platform.sh` — `platform()`, `system_pkg_missing`, `system_pkg_install` (cross-platform plans 02–03)
- `lib/packages.sh` — `meta_deps` map resolution (exact platform → `linux`)
- `package.yml` files with a `system` key across ai, pde, pdt, user (`grep -l '^system:' */packages/*/package.yml`)
- https://docs.brew.sh/Homebrew-on-Linux — Fedora prerequisites

## Overview

Add Fedora as a supported platform: detection, brew prerequisites, a dnf adapter, and `system.fedora` entries where package names differ from Debian. Test on Fedora VMs built by PIM with kickstart.

## Implementation

### 1. Detection

`platform()` (installer and `lib/platform.sh`) recognizes `ID=fedora`, and `ID_LIKE` containing `fedora` or `rhel` (Rocky, Alma, RHEL) → `fedora`.

### 2. Prerequisites

`sudo dnf group install -y development-tools` plus `procps-ng curl file git`. Check first with `rpm -q` (and `dnf group list --installed` for the group) so a configured machine does not prompt.

### 3. `system` adapter

- `system_pkg_missing`: `rpm -q <name>` per package (non-zero → missing); `@group` names via `dnf group list --installed`
- `system_pkg_install`: `sudo dnf install -y <names…>`

### 4. Package entries

For each `system` map, add `fedora:` where the names differ, or rely on `linux:` where they match. Known differences to check:

| debian | fedora |
|---|---|
| `build-essential` | `@development-tools` |
| `nfs-kernel-server` | `nfs-utils` |
| `libssl-dev` | `openssl-devel` |
| `libncurses-dev` | `ncurses-devel` |
| `libreadline-dev`, `libyaml-dev`, `zlib1g-dev` | `readline-devel`, `libyaml-devel`, `zlib-ng-compat-devel` |
| `qemu-system-*` | `qemu-system-*` / `qemu-kvm` |
| `bridge-utils` | not packaged in recent Fedora; use `iproute` (`ip link`) |
| `xz-utils`, `dosfstools`, `exfat-fuse` | `xz`, `dosfstools`, `exfatprogs` |

Services: Fedora enables SELinux and firewalld by default — packages that open ports or mount (nfs, dnsmasq, pcs bridges) may need `firewall-cmd` / SELinux booleans in their `install_linux` hooks. Record each case in `log.md`.

### 5. Login shell

`chsh` is not installed by default on Fedora; add `util-linux-user` to the zsh package's `system.fedora`.

## Test Spec

Fresh Fedora VM (current release), kickstart-built:

- Admin user: `install.sh` → one sudo prompt, brew owner, ppm + zsh installed; second run prompts for nothing
- Second non-admin user: consumer mode, no sudo
- Install each package that declares `system`; verify services start where applicable
- Rocky Linux VM (opportunistic): `platform()` returns `fedora`; bootstrap succeeds or failures are recorded

## Verification

- [ ] `platform()` returns `fedora` on Fedora and Rocky
- [ ] dnf adapter installs only missing packages
- [ ] Every distro-keyed `system` map resolves on Fedora or the package documents why it is unsupported
- [ ] SELinux/firewalld adjustments recorded per package in `log.md`
