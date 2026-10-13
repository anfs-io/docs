---
---

# Plan 01 — `lib/anfs/link.sh`, and stow out of ppm, wsm and the bootstrap

## Context — read these files first

- `units/linker.md` — the table of every stow call and what replaces it
- `packages/ppm/home/.local/lib/ppm/installer.sh` — `stow_package`, `stow_subdir`,
  `unstow_package`, `PPM_IGNORE_ARGS`, `_stow_ignore_arg`, `is_stow_ignored`,
  `remove_dangling_links`, `force_remove_conflicts`, `_backup_file`, `meta_cleanup_stale`,
  `_link_points_into`. This file is in the bash 3.2 bootstrap set
- `packages/ppm/home/.local/lib/ppm/file.sh` — `add`/`claim` (line 200), `reset` (line 259)
- `packages/dev/home/.local/lib/ppm/move.sh` — `_move_restow`, `_move_rollback_stow`
- `packages/wsm/home/.local/bin/wsm` — `_stow`, `WSM_STOW_IGNORE`, `_stow_ignores`,
  `_unit_files`, `_relpath`, `_link_target`, the install and remove paths, `add`
- `install.sh` — the bootstrap formulas, the pre-pass before `stow_package`, the stow checks
- `packages/anfs/tests/bash.bats` — how the bootstrap set is held to bash 3.2
- `packages/ppm/tests/backup.bats`, `packages/wsm/tests/install.bats`

## Overview

One library links files. ppm, wsm and `install.sh` call it where they call stow, and the code
that exists only to keep stow from aborting is deleted. Behaviour a user sees changes in three
places: a conflict report lists every file, `README.*` and `.gitignore` under `home/` are linked,
and stow is no longer installed.

## Implementation

### 1. The library

`packages/anfs/home/.local/lib/anfs/link.sh`, bash 3.2, functions only. Paths are relative to the
`from` directory; lists are newline-separated on stdin or in a file (no associative arrays).

```
anfs_link_plan   <from> <to> <files-file> [skip-file]   classify every target; print the plan; 1 on any conflict
anfs_link_apply  <from> <to> <files-file> [skip-file]   plan, then make the links; nothing is touched on a conflict
anfs_unlink      <from> <to> <files-file>               remove each link that points into <from>; leave anything else
anfs_unlink_tree <from> <to>                            the same for every file under <from> (no tracker needed)
anfs_link_one    <from-file> <to-path>                  one relative link, parents made
anfs_link_owner  <path>                                 "<sources-root-relative target>" when it is a link into $ANFS_SOURCES_HOME, else 1
anfs_relpath     <from-dir> <to>                        wsm's _relpath, moved here
```

- A link is relative, from the link's directory to the file, exactly as `stow --no-folding`
  writes it. Directories on the way are real and are created; they are never links.
- The plan prints one line per file: `link`, `keep` (already ours and current), `replace` (ours,
  dangling, or pointing at another file of the same `from`), `skip` (in the skip set), `other`
  (a live link into another directory under `$ANFS_SOURCES_HOME`: another layer's), `conflict`
  (a file, a directory where a file should be, or a link elsewhere).
- `other` and `conflict` are both returned to the caller; what to do with them is the tool's
  (ppm backs up on `-f`; wsm refuses).
- Names are handled as data: spaces, `+`, `(`, `[`, a leading `-`. A name with a newline is
  refused with a message, once, by the plan.
- A directory in the target path that is itself a link is a conflict unless it resolves inside
  `<to>`.
- The only name the library skips by itself is `.DS_Store`.

### 2. ppm

- `stow_subdir` builds the file list (`package_links`), runs the plan with the skip set, applies
  `-f` to what the plan returned (`_backup_file` for `conflict`, delete for a link into the
  sources), then `anfs_link_apply`. `remove_dangling_links`, `force_remove_conflicts` and the
  stale pass of `meta_cleanup_stale` become cases of the plan and are removed as separate walks.
- `PPM_IGNORE_ARGS` becomes `PPM_SKIP_FILE`, a temp file of exact paths: seeded from
  `protected.yml`, appended with each layer's files. `_stow_ignore_arg` and `is_stow_ignored` go;
  `is_stow_ignored` callers use `grep -qxF`.
- `unstow_package` is `anfs_unlink` with the tracker's files, `anfs_unlink_tree` when there is
  no tracker.
- `file.sh`: `add`, `claim` and `reset` link the one file (`anfs_link_one`) instead of
  re-stowing a package. `reset`'s rebuilt ignore list goes.
- `move.sh`: `_move_restow` links the tracker's list from the new directory.
- The conflict message keeps its wording and now lists every file.

### 3. wsm

- `_stow`, `WSM_STOW_IGNORE`, `WSM_STOW_HOME` and `_stow_ignores` are removed. Install links
  `_unit_files` of the level with `anfs_link_apply`; remove is the existing loop over the
  recorded files. `_relpath` is the library's.
- `wsm implode` removes a leftover `$WSM_STATE_HOME/stow/` from an older version.
- The stale-link case (an absolute link from wsm 0.3) is the plan's `replace`.

### 4. The bootstrap

- `install.sh` stops installing and requiring `stow`; the pre-pass before `stow_package` goes
  (the plan's `keep` and `replace` cover it). `link.sh` is sourced from the clone with the other
  bootstrap libraries and added to `PPM_BOOTSTRAP_LIBS`' check in `bash.bats`.
- `packages/ppm/package.yml` and anything else that declares or checks for stow.
- The round trip's bootstrap snapshot (`brew install stow yq mise`) drops stow, which makes the
  round trip the proof that nothing needs it.

### 5. Documentation

CLAUDE.md: "Dependencies", `install.sh`'s paragraph, the wsm bullet on stow's ignore list, the
`ppm move` and `ppm file` notes that describe stow's behaviour, "Lib Structure" (`link.sh`, and
the bootstrap set). The words "stow" and "stowed" stay where they mean "linked into place";
`ppm` help text keeps them.

## Test Spec

`packages/anfs/tests/link.bats` (temp dirs only):

- Plan classes: each of `link`, `keep`, `replace`, `skip`, `other`, `conflict`, and a run with
  two conflicts reports both and changes nothing.
- Names: spaces, `+`, `(`, `[`, a leading `-`, a dot file; a newline is refused.
- A directory in the path that is a link out of the target is a conflict; one that resolves
  inside is followed.
- Links are byte-identical to what `stow --no-folding` writes for the same tree (skipped when
  stow is not installed), and `stow -D` removes what the library linked.
- `anfs_unlink` leaves a file, a link elsewhere and a link into another package; `_tree` works
  with no list.
- `link.sh` parses under `/bin/bash` (`bash.bats`).

ppm: every existing suite passes unchanged; new cases for a layered package (the higher layer's
file is skipped by exact name, including a name with `+`), `README.md` under `home/` is linked,
an install made with stow links is taken over with no link rewritten (compare inodes), and
`backup.bats` still holds.

wsm: every existing suite passes with no `stow` on PATH.

## Verification

- [ ] `bats packages/{anfs,ppm,wsm}/tests </dev/null`, with `stow` removed from PATH
- [ ] `roundtrip debian` from a bootstrap snapshot that has no stow; the `$HOME` diff is clean
- [ ] On a copy of this Mac's `$HOME` state (47 packages, 224 links): `ppm install -n` for every
      installed package plans only `keep`
- [ ] `roundtrip-macos` (a person, or when the tart box is free)
