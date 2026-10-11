---
---

# Plan 02 — Arch

> **On hold.** Fedora and RHEL derivatives cover multi-family Linux support for now. Before resuming: the official `archlinux` container image is amd64-only, so test on an x86_64 host or VM (or an unofficial Arch Linux ARM image).

## Context — read these files first

- `lib/platform.sh` — adapters as extended for Fedora in plan 01
- `install.sh` — `platform()` and prerequisite bootstrap
- `units/distros/01-fedora/log.md` — package name mapping and service issues found for Fedora
- https://docs.brew.sh/Homebrew-on-Linux — Arch prerequisites

## Overview

Add Arch following the Fedora pattern. Decide during this plan how much Arch should lean on brew versus pacman: Arch repositories carry most CLI tools at current versions, so brew adds less here. The default stays the cross-platform model (brew for CLI tools) unless testing shows a concrete problem.

## Implementation

### 1. Detection

`ID=arch`, and `ID_LIKE` containing `arch` (EndeavourOS, Manjaro) → `arch`.

### 2. Prerequisites

`sudo pacman -S --needed --noconfirm base-devel procps-ng curl file git`. Check with `pacman -Q` first so a configured machine does not prompt.

### 3. `system` adapter

- `system_pkg_missing`: `pacman -Q <name>`; groups such as `base-devel` via `pacman -Qg <group>`
- `system_pkg_install`: `sudo pacman -S --needed --noconfirm <names…>`
- AUR packages are out of scope; a package needing AUR software documents it and uses brew or mise instead

### 4. Package entries

Add `arch:` to every distro-keyed `system` map where names differ from `linux:`, starting from the Fedora mapping in plan 01's log. Arch ships headers inside the main packages (no `-dev`/`-devel` split), so most build-dependency lists shrink to `base-devel` plus the library names.

### 5. Rolling release

Pin test VM images by date and record the image date in `log.md`, so failures can be traced to package updates.

## Test Spec

Fresh Arch VM (archinstall config, dated image):

- Admin: `install.sh` → one sudo prompt, brew owner, ppm + zsh; re-run prompts for nothing
- Second non-admin user: consumer mode
- Install each package with a `system` declaration; verify services where applicable

## Verification

- [ ] `platform()` returns `arch` on Arch and derivatives
- [ ] pacman adapter installs only missing packages
- [ ] Every distro-keyed `system` map resolves on Arch or the package documents why it is unsupported
- [ ] Brew vs pacman decision recorded in `log.md`
