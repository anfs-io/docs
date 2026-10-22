---
---

# Plan 03 — `anfs owner`, `anfs mv`, `anfs status`

## Notes from the 2026-10-21 review — to settle before this plan is refreshed

- **2026-10-22: `ppm move` is settled** — it becomes `linker/05-move` (each tool's own `move` on `lib/anfs/move.sh`); `anfs mv <source/name> --to <source>` here only runs the owning tool's `move`.

- **2026-10-22: the opt-in shell `mv` wrapper (section 5) is out of this plan**, moved to the end of the roadmap ("Last").

The plan below predates wsm-spaces and plan 02's rewrite. The review moved its theme to **file
operations, shared**; these notes say where it is going. The rest of the plan is refreshed only
after they are settled with the person.

- **A shared `file` command group, built like config's** (`units/config.md`, plan 02: a library
  plus a group declared through `lib/anfs/cli.sh`). The library is `lib/anfs/files.sh`, which
  linker 02 creates for copies (copy-if-missing) and designs as this group's home.
  - `ppm file add|claim|reset|protect|unprotect`
  - `wsm file add|protect|unprotect`. `claim` and `reset` only where layers exist: ppm's; spaces
    do not layer.
  - **`wsm add` goes, with no alias**: it becomes `wsm file add`.
- **Each tool supplies an adapter** the group calls: the owner of a path and its source directory;
  its layers (ppm only); its record (ppm's tracker; wsm has none since wsm-spaces 01, so a move in
  a space is the file's move in the source plus a relink); its protected list (ppm's
  `protected.yml`; wsm's new `~/.config/wsm/protected.yml`). Tools that do not link content into
  `~` (pcm, pmm, ovm) do not get the group.
- **`anfs owner|mv|status` sit on top**, dispatching through the adapters instead of the
  `ppm __file` / `wsm __file` plumbing of section 1 below.
- **ppm's `file.sh` shrinks to its adapter**; the command logic moves into `files.sh`.
- **`ppm move`'s fate** (section 3a: maybe gone, in favour of the shell `mv` wrapper) is still to be
  discussed.
- **It must adapt to wsm having no tracker** (wsm-spaces decisions 4, 6, 18). The mentions of
  `wsm status` below (section 4's "wsm status ... supplies the rows for spaces", the Context's
  `cmd_status`) and of wsm's trackers are stale: spaces' rows come from `wsm list --json`'s
  per-file detail.
- From plan 02: `anfs owner` of a path *inside* a directory link (a file of the source, not a link
  itself) must still name the package (the nearest ancestor that is a link into the sources); the
  group's commands refuse such a path as plan 02's `ppm file` already does. `files:` copies are the
  user's: `anfs owner` names nobody, and `status` does not report them as drift.

## Context — read these files first

The job touches core and anfs-dev: `anfs work new <job> core anfs-dev`.

- `units/linker.md`, section 3; the logs of plans 01 and 02
- anfs-dev `packages/dev/home/.local/lib/ppm/move.sh` — `ppm move` (`cmd_move`, `_move_restow`,
  `_move_rollback_stow`), which 3a folds into ppm
- `packages/ppm/home/.local/lib/ppm/file.sh` — `add`, `claim`, `reset`, `protect`; `claims.yml`,
  `protected.yml`, the trackers
- `packages/wsm/home/.local/bin/wsm` — `cmd_add`, `cmd_status`, the trackers
- `packages/anfs/home/.local/bin/anfs`; `packages/anfs/home/.config/sh/anfs.sh`

## Overview

Whether a file under `$HOME` is a package's, a space's or the user's own can be read from the
file. This plan gives that one command, gives moving such a file one command, and reports where
the home directory and the sources have drifted apart.

## Implementation

### 1. Plumbing in the tools

anfs dispatches; the tool that owns a kind does the work. Each gets two plumbing commands
(`cmd_` functions, not in help), with `--json` answers:

```
ppm __file owner <path>          {tool, source, name, file, tracked}
ppm __file mv <from> <to>        move within the package; relink; tracker and claims follow
wsm __file owner <path>
wsm __file mv <from> <to>
```

`mv` within a package: the file moves under `home/` to the path that corresponds to `<to>`, the
link is made there, the old link is removed, and the tracker's entry is renamed. Moving out of
`$HOME` (or out of the space's tree) is refused: that is `protect`, or a copy.

### 2. `anfs owner <path>...`

A table through `anfs_table`: `path`, `tool`, `source`, `name`, `file` (the path in the source).
A path nothing owns has a null tool. A real file that a tracker lists is shown as owned with
state `replaced` (an editor wrote over the link).

### 3. `anfs mv <from> <to>` and `anfs mv <from>... --to <source/package>`

| From | To | Runs |
| --- | --- | --- |
| owned | a path with the same owner's reach | the owner's `__file mv` |
| not owned | `--to <source/package>` | `ppm file add` |
| not owned, inside an installed space's tree | — (`anfs mv --adopt <file>`) | `wsm add` |
| owned by a package | `--to <source/package>` | `ppm file claim` |
| a package (`<source>/<package>`) | `--to <source>` | `ppm __package mv` (below) |

It prints what changed in the source (`moved packages/git/home/.gitconfig -> ...; not
committed`). It never commits. `-n` prints the plan.

### 3a. A whole package — to be discussed

`ppm move` (anfs-dev's `move.sh`) is broken from plan 01 on. What becomes of it is open
(2026-10-21): it may go entirely, in favour of the shell `mv` wrapper (section 5), or become
`anfs mv <source/package> --to <source>` with the work in ppm. Its tests are part of the same
discussion (it has no bats file today). Settle this before the plan is run.

### 4. `anfs status`

Drift, one row each, with the fix in the last column:

- a tracked file that is now a real file (`replaced`): `anfs mv --adopt` back into the source,
  or `ppm file protect`
- a tracked link that is gone, with an untracked link elsewhere pointing at the same file
  (`moved`): `anfs mv <old> <new>`
- a dangling link into the sources (`dangling`): reinstall the package
- a tracked path with nothing there (`missing`): reinstall

It reads the trackers and the links only (no git, no network). `wsm status` stays as wsm's own
question and supplies the rows for spaces.

### 5. The shell wrapper (opt-in)

`sh/anfs.sh` defines `mv()` only when `ANFS_SHELL_MV=true`. For each non-option argument but the
last it does one `readlink`; if none is a link into `$ANFS_SOURCES_HOME`, it runs
`command mv "$@"`. Otherwise it runs `anfs mv` with the same arguments, which refuses option
combinations it does not understand and says to use `command mv`. Interactive shells only, as
all of `sh/` is. No `rm`, no `cp`.

### 6. Documentation

CLAUDE.md "Claiming and Protecting Files": `anfs mv` and `anfs owner` as the front door, the
tool commands as what they call. The key table rows for `anfs owner` and `anfs status`.

## Test Spec

`packages/anfs/tests/files.bats` with a fixture source (a package with layers, a space):

- `owner`: a package's file, a space's file, the user's own, a replaced one; `--json` keys.
- `mv` within a package to a deeper directory: the source file moved, the link is relative and
  resolves, the tracker has the new path, `ppm install` afterwards changes nothing.
- `mv` of an unowned file with `--to`: as `ppm file add`. A protected file is refused.
- `status`: each of the four states, and empty after the fix it names.
- The wrapper: an unowned `mv a b` reaches `command mv` with its arguments untouched (including
  `-n` and a name with spaces); an owned one reaches `anfs mv`.

## Verification

- [ ] `bats packages/{anfs,ppm,wsm}/tests </dev/null`
- [ ] On an isolated copy of a real install: `anfs status` is empty; save a linked file with an
      editor that writes by rename, and it shows `replaced`
- [ ] `anfs owner ~/.zshrc` names `stack/zsh` (or the layer that wins)
