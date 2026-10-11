---
---

# Plan 03 — Declarative Dependencies

## Context — read these files first

- `lib/installer.sh` — `install()`, `install_single_package()`, `remover()`
- `lib/packages.sh` — `meta_depends()`, `meta_version()`, `expand_packages()`, tracker functions (`meta_mark_installed`, …), `show()`
- `lib/core.sh` — `install_dep()` (kept as a deprecated wrapper until plan 04 finishes)
- `lib/platform.sh` — from plan 02: `platform()`, `brew_prefix()`, `brew_is_owner()`, `brew_require_owner()`
- `chorus/units/cross-platform.md` — decisions (three tiers, owner model, no automatic system package removal)

## Overview

Packages declare what they need in `package.yml` instead of calling `install_dep` from hooks:

```yaml
version: 0.1.0
author: rjayroach
depends: [mise]
platforms: [macos, linux]      # optional; omit = everywhere

brew: [tmux, bat]              # list: every platform
cask: [ghostty]                # macOS only

system:                        # map when names differ per platform
  debian: [nfs-kernel-server]
  fedora: [nfs-utils]          # added in the distros unit
```

A map lets a package use different managers per platform:

```yaml
# podman
brew:
  macos: [podman, podman-compose]
system:
  linux: [podman, podman-compose]
```

`ppm install` gathers the declarations of every resolved package, installs what is missing in one batch per manager, then runs the existing per-package lifecycle. After stowing, it runs one `mise install` if any installed package ships mise config. `ppm remove` uninstalls the brew formulas and casks ppm installed for that package when nothing else needs them.

## Implementation

### 1. Resolving declarations (`lib/packages.sh`)

- `meta_platforms <pkg_dir>` — `.platforms[]`
- `meta_supported <pkg_dir>` — true when `platforms` is absent, or contains `platform()`, or contains `linux` and the platform is a Linux distro
- `meta_deps <pkg_dir> <key>` — for `brew`, `cask`, `system`:
  - list (`yq '.<key> | type'` is `!!seq`) → its items
  - map (`!!map`) → items under the exact platform key, else under `linux` when on Linux, else nothing
  - For `system` only: if the map has keys for other Linux distros but neither the current distro nor `linux`, return non-zero so the caller fails with `no system packages declared for <platform>` instead of silently skipping

Each prints one name per line.

### 2. Platform support

- `install()`: after resolving, fail before any install work if a resolved package is not `meta_supported`, naming the package and its `platforms`.
- `expand_packages()` (`ppm install repo/`): skip unsupported packages with a note instead of failing.
- `list`: unchanged (lists everything); `show` prints `Platforms:`.

### 3. Package manager adapters (`lib/platform.sh`)

- `system_pkg_missing <names…>` / `system_pkg_install <names…>` — debian: `dpkg-query -W -f='${Status}'` and `sudo apt-get install -y`. Dispatch on `platform()` so the distros unit only adds dnf/pacman cases.
- `brew_missing <names…>` / `brew_install <names…>` — `brew list --formula -1` / `brew install`; tap-qualified names (`dmno-dev/tap/varlock`) compare by final segment
- `cask_missing <names…>` / `cask_install <names…>` — `<prefix>/Caskroom/<name>`; Homebrew installs casks on Linux as well (verified for `claude-code` and `1password-cli`), so a macOS-only GUI cask belongs under a platform map in the package

Installing only what is missing keeps re-runs free of sudo prompts and network work. No upgrades (unit Out of Scope).

### 4. Batch install in `install()` (`lib/installer.sh`)

After `RESOLVE_ORDER`/`RESOLVE_DIRS` are known and platform support is checked, before the per-package loop:

1. Collect the union of `system`, `brew`, `cask` across `RESOLVE_DIRS` (first-seen order; bash 3.2 compatible, no associative arrays).
2. `system`: compute missing; if any, print them, then `system_pkg_install` (sudo prompts here, once). If sudo is unavailable, `ppm_fail` naming the packages and stop.
3. `brew`, then `cask`: compute missing; owner → install in one call each; consumer → `brew_require_owner "brew install <missing>"` and stop.
4. Remember which names were actually installed in this run, per package, for the tracker (step 6).

`-c` (config only) skips this step, like it skips hooks. `-s` still installs the declarations of the packages it installs.

### 5. One `mise install`

After the per-package loop, if any resolved package stowed a file under `.config/mise/`, run `mise install` once (skipped with `-c`). Failure is a `ppm_fail`, not an abort — stowed config is still correct.

### 6. Tracker records what ppm installed

`meta_mark_installed` writes, in addition to `files`:

```yaml
installed_deps:
  brew: [bat]        # only names that were missing and installed by this run
  cask: []
```

Names already present before the run are not recorded, so ppm never uninstalls software the user installed themselves. On reinstall (`-r`), carry forward the previous `installed_deps`.

### 7. Removal

In `remover()`, after `post_remove` and before `meta_mark_removed`:

- For each name in this package's `installed_deps.brew`/`cask`: uninstall it if no other tracker lists it in `installed_deps` and no other installed package declares it (via `meta_deps`).
- Owner only; consumers skip with a note.
- Never uninstall `system` packages; if the package declared any, list them in a `user_message` so the user can remove them manually.
- Uninstall failures (e.g. still required by another formula) are reported, not fatal.

### 8. Visibility

- `ppm show` prints `Platforms:`, `Brew:`, `Cask:`, `System (<platform>):` for the resolved values.
- `ppm install --debug` logs each manager's missing and installed lists.

### 9. Deprecate `install_dep`

Keep `install_dep` working, but `debug` a deprecation note pointing at `package.yml`. Plan 04 removes it after migration.

### 10. Docs

`CLAUDE.md` "package.yml" section: document `platforms`, `brew`, `cask`, `system` (list vs map, lookup order), the owner model, and that hooks should not install software.

## Test Spec

Scratch repo with test packages in the Debian 13 and macOS VMs:

- Owner installs a package with a missing formula → formula installed once; tracker lists it under `installed_deps.brew`
- Install again → no brew call, no sudo prompt
- Two packages declaring the same formula; remove one → formula stays; remove the other → formula uninstalled
- Formula installed manually beforehand; install and remove a package declaring it → formula not uninstalled
- Consumer with a missing formula → clear failure naming formula and owner; nothing written to the prefix
- Missing system package → one sudo prompt for the batch; present → no prompt
- `system` map with only `fedora` on Debian → clear failure; with `linux` → used on Debian
- podman-style map (`brew.macos` + `system.linux`) → brew on macOS, apt on Debian
- `platforms: [macos]` on Debian → `ppm install` refuses; `ppm install repo/` skips it with a note
- `-c` → no package-manager calls
- Package stowing `.config/mise/conf.d/x.toml` → one `mise install` after all stows

## Verification

- [ ] `meta_platforms`, `meta_supported`, `meta_deps` exist and are used by `install()`, `expand_packages()` and `show`
- [ ] Batch install happens before hooks; hooks can rely on declared tools
- [ ] Trackers record only ppm-installed names; removal honors shared declarations
- [ ] No automatic system package removal
- [ ] bash 3.2 compatible (no associative arrays)
- [ ] All Test Spec cases recorded in `log.md`
