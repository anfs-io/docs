---
---

# Plan 02 — Per-platform files declared in `package.yml`

## Context — read these files first

- `units/linker.md`; plan 01's `log.md`
- `packages/ppm/home/.local/lib/ppm/installer.sh` — `stow_package` (`for subdir in home
  ${PPM_GROUP_ID:-}`), `unstow_package`
- `packages/ppm/home/.local/lib/ppm/packages.sh` — `PPM_CORE_KEYS`, how `brew`/`system` read a
  list or a per-platform map (the same reader is used here)
- `install.sh` — where `PPM_GROUP_ID` is written to `anfs.local.conf`
- The packages that have such a directory today: stack `op` (`macos/`, `linux/`), user `host`
  (`macos/`), user `ssh` (`macos/`)

## Overview

A package's files for one platform live in a sibling of `home/` named for the platform
(`macos/`, `linux/`), linked when `PPM_GROUP_ID` in `anfs.local.conf` says so. That is a second
tree to look in, a setting that must match the machine, and no way to say "Debian only".

After this plan a package has one tree, `home/`, and `package.yml` says which of its paths are
for which platform. No file in any source today exists in both `home/` and a platform directory,
so nothing needs two versions of one path.

## Implementation

### 1. The declaration

```yaml
stow:
  macos: [.ssh/config.d/99-op.conf, Library/]
  linux: [.config/mise/conf.d/op.toml]
  debian: [.config/apt/]
```

- A path is relative to `home/`. One ending in `/` is everything under that directory.
- A path listed under a platform is linked only there. Everything not listed is linked
  everywhere. Keys are `platform()` values and `linux`, as for `brew` and `system`; a file is
  linked when any key it is listed under matches this machine.
- `stow` joins `PPM_CORE_KEYS`.
- `ppm validate` (or install, where there is no validate) refuses a listed path that does not
  exist under `home/`, and a key that is not a platform.

### 2. ppm

- `stow_package` links `home/` only. The files of other platforms go into the skip set before
  the plan (plan 01's skip file), so they are neither linked nor conflicts.
- `ppm show` lists a package's per-platform files with their platform.
- The tracker records what was linked, as now; a file whose platform does not match is not in it.
- `PPM_GROUP_ID` is removed: from the installer, from `install.sh`'s seed of `anfs.local.conf`,
  and from `anfs.conf`'s documentation. A leftover line in an existing `anfs.local.conf` is
  ignored.
- A package that still has a `macos/` or `linux/` directory gets one warning naming this key.

### 3. The packages

Move each platform directory's files into `home/` and add the `stow:` map: stack `op`; in the
user's repo `host` and `ssh` (on a branch; these are the user's own files, so only the move and
the key, nothing else). A link that pointed into `macos/` is replaced on the next install
(the plan's `replace`).

### 4. Documentation

CLAUDE.md "Package Structure": the `stow:` key beside `brew`/`cask`/`system`. Remove
`PPM_GROUP_ID` from "Configuration".

## Test Spec

`packages/ppm/tests`:

- A file under `macos:` is linked on macOS and absent on Debian (stub `platform`); `linux:`
  matches Debian and Fedora; `debian:` only Debian; a directory entry covers what is under it.
- A listed path that does not exist is refused; an unknown key is refused.
- A package installed with a `macos/` directory by the old ppm, then converted: reinstall leaves
  one link per file, pointing into `home/`, and the tracker matches.
- Layers: a higher layer's file still wins over a lower layer's per-platform file of the same path.

## Verification

- [ ] `bats packages/ppm/tests </dev/null`
- [ ] `ppm install -n stack/op` on this Mac plans `replace` for its one macOS file and nothing else
- [ ] `roundtrip debian`: `op`'s Linux file is linked, its macOS file is not
- [ ] `grep -rn PPM_GROUP_ID` in core and stack finds nothing
