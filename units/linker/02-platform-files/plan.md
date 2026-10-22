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

### 3a. stack's bash comment

`packages/bash/install.sh`'s header says the distro's `~/.bashrc` is moved aside "instead of
letting stow abort the install". After plan 01 nothing aborts; the conflict is reported. Reword
the reason (the user keeps their file rather than `-f` deleting it); the hook stays.

### 3b. Directories linked whole: `link_dirs:` (decided 2026-10-21)

A package may name directories under `home/` that are linked as one link to the directory,
instead of file by file:

```yaml
link_dirs: [.local/lib/anfs]
```

- Only when the package says so; ppm never folds on its own (stow's folding is what
  `hooks.bats`' "when stow folded the hook directory" was about).
- Why: a tool's library set is then always whole: a file added or removed upstream is there after
  `git pull`, with no reinstall, and no lib can be half-linked. The fallbacks to the core clone's
  `link.sh` in ppm and wsm are deleted.
- The plan treats such a directory as one entry (`keep`, `link`, or a `conflict` naming any file
  another package put there). The tracker records the one path. A higher layer cannot override
  one file inside it, and `ppm file protect` refuses a path inside it.
- Only for a directory one package owns. `.local/lib/ppm` stays per-file (core/ppm, anfs-dev/dev
  and stack/ruby all write there). Converted here: core `anfs` (`.local/lib/anfs`), `pcm`
  (`.local/lib/pcm`), `pmm` (`.local/lib/pmm`), stack `tailscale` (`.local/lib/tsm`), stack
  `rpi-imager` (`.local/lib/rpi-imager`) — check each has one owner first.
- No migration code: the per-file links on this Mac are replaced by hand after merge
  (`ppm install -f` of each converted package, or remove the old links first); the log lists it.

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
