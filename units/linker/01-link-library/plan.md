---
---

# Plan 01 — `lib/anfs/link.sh`, and stow out of ppm, wsm and the bootstrap

Refreshed 2026-10-21 against main

## Context — read these files first

The job is `anfs work new <job> core anfs-dev`: core for the work, anfs-dev only for the two
stow dependencies in its tests (section 4). Paths below are core's unless they name another source. `ppm move` (anfs-dev's `packages/dev/home/.local/lib/ppm/move.sh`) is plan 03's.
It calls `unstow_package`, `PPM_IGNORE_ARGS`, `_stow_ignore_arg` and `stow`, so it is broken from
this plan until plan 03 decides its fate (it may go, in favour of the shell `mv`). Accepted
(2026-10-21): no shims; `log.md` says so.

- `units/linker.md` — the table of every stow call and what replaces it
- `packages/ppm/home/.local/lib/ppm/installer.sh` — `stow_package`, `stow_subdir`,
  `unstow_package`, `PPM_IGNORE_ARGS`, `_reset_ignore_args`, `_stow_ignore_arg`,
  `is_stow_ignored`, `package_links`, `remove_dangling_links`, `force_remove_conflicts`,
  `_backup_file`, `meta_cleanup_stale`, `_link_points_into`; `STOWED_FILES`, which
  `install_single_package` hands to `meta_cleanup_stale`, `_reload_stowed_libs` and
  `meta_mark_installed`. This file is in the bash 3.2 bootstrap set
- `packages/ppm/home/.local/lib/ppm/file.sh` — `_file_claim` (`add` and `claim`, stow at line
  200), `_file_reset` (the rebuilt ignore list at 251–257, stow at 259–260)
- `packages/ppm/home/.local/lib/ppm/implode.sh` — `_implode_sweep_links` (a caller of
  `_link_points_into`), `_implode_prune_dirs`, the "Kept: Homebrew, stow, ..." line
- `packages/wsm/home/.local/bin/wsm` — `WSM_STOW_HOME`, `WSM_STOW_IGNORE`, `_stow`,
  `_stow_ignores` (`STOW_IGN`), `_unit_files`, `_relpath`, `_link_target`; install is
  `_link_unit` (called by `_install_unit`), remove is `_unlink_unit` (called by `cmd_remove`,
  `_unlink_orphans`, `cmd_implode`), `wsm add` is `_add_one`
- `install.sh` — `base_formulas`, `stow_base` (the stow check, the pre-pass, `stow_package`), the
  stow check after `setup_brew` in `main`, `source_libs` and `PPM_BOOTSTRAP_LIBS`
- `packages/anfs/tests/bash.bats` — "the bootstrap set parses under the older bash" and "the
  bootstrap set is what install.sh sources": how the bootstrap set is held to bash 3.2
- `packages/ppm/tests/backup.bats`, `packages/wsm/tests/install.bats`, `manage.bats`
- `AGENTS.md` (core's; `CLAUDE.md` is `@AGENTS.md`)

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
  `STOWED_FILES` keeps its three readers in `install_single_package`.
- `PPM_IGNORE_ARGS` becomes `PPM_SKIP_FILE`, a temp file of exact paths: seeded from
  `protected.yml` (today `_reset_ignore_args`, also called by `install.sh`'s `stow_base`),
  appended with each layer's files. `_stow_ignore_arg` and `is_stow_ignored` go;
  `is_stow_ignored` callers use `grep -qxF`.
- `unstow_package` is `anfs_unlink` with the tracker's files, `anfs_unlink_tree` when there is
  no tracker.
- `file.sh`: `_file_claim` (`add`, `claim`) and `_file_reset` link the one file
  (`anfs_link_one`) instead of re-stowing a package. `_file_reset`'s rebuilt ignore list goes.
- `implode.sh`: `_implode_sweep_links` matches links into the sources with `anfs_link_owner`,
  the one answer to "is this ours"; `_link_points_into` goes if nothing else calls it. "Kept: Homebrew, stow, yq, mise" drops stow.
- The conflict message keeps its wording and now lists every file.
- `ppm install -n`: run the plan for each package and print it (`link`, `keep`, `replace`,
  `skip`, `other`, `conflict`, and what `-f` would back up), changing nothing: no links, no
  hooks, no tracker, no brew. The plan pass makes it nearly free. The other commands that
  install, add, remove or purge get `-n` in plan 04.
- Callers that rely on `-f` and the backup, unchanged: `anfs onboard`'s user source and
  `install.sh --repo` (`ppm install -f <alias>/anfs` over the plain `user.list`, `onboard.sh`,
  `install_repo`), stack/git's onboarding (`ppm file add`).

### 3. wsm

- `_stow`, `WSM_STOW_IGNORE`, `WSM_STOW_HOME`, `_stow_ignores` and `STOW_IGN` are removed.
  `_link_unit` links `_unit_files` of the level with `anfs_link_apply`; its own conflict walk
  becomes the plan. `_unlink_unit` is the existing loop over the recorded files, without the
  `_stow -D` before it. `_relpath` is the library's (`_add_one`'s two `ln -s`).
- `wsm implode` removes a leftover `$WSM_STATE_HOME/stow/` from an older version (it already
  deletes all of `$WSM_STATE_HOME`).
- The stale-link case (an absolute link from wsm 0.3, `_link_unit`) is the plan's `replace`.
- The `EX_UNAVAIL` "wsm needs GNU stow" check goes; the README's exit code `69` drops stow.
- pvm's links into wsm levels (`pvm link`, `_link_make`) are not in any tracker and are left
  alone by remove, as today.

### 4. The bootstrap

- `install.sh` stops installing and requiring `stow`: `base_formulas`, the check at the top of
  `stow_base`, the `yq`-and-`stow` check after `setup_brew` in `main`. `stow_base`'s pre-pass
  (the loop before `_reset_ignore_args`/`stow_package`) goes; the plan's `keep` and `replace`
  cover it. The header comments, `--script-only`'s "(needs stow)" and the `brew_env` comment
  follow.
- `link.sh` is sourced with the other bootstrap libraries. `source_libs` reads only
  `PPM_LIB_SUBDIR` today (from the checkout, the clone, or downloaded when piped), so it gains
  `packages/anfs/home/.local/lib/anfs/link.sh` in all three. `bash.bats`: add it to the parse
  list and to the "is what install.sh sources" check.
- Nothing declares stow as a dependency: not `packages/ppm/package.yml`, not
  `stacks/system/stack.yml`. `packages/anfs/package.yml`'s comment ("like stow and yq it is base
  software") drops stow.
- Machines that have stow keep it: nothing uninstalls the formula (decided 2026-10-21).
- `anfs implode --all` removes the formulas `install.sh` recorded
  (`~/.local/state/anfs/bootstrap`), so an older install still loses stow there; a new one never
  records it.
- In anfs-dev (this job's second repo): the round trip's bootstrap snapshot
  (`packages/dev/tests/roundtrip`, `build_bootstrap`: `brew install -q stow yq mise`) drops stow,
  and anfs-dev `AGENTS.md`'s line on it follows; `work.bats`' `docs_space` loses its "no stow"
  skip. Nothing else in anfs-dev changes here (`move.sh` and its prose are plan 03's). The
  snapshot is built once and cached (`localhost/anfs-test:<distro>-bootstrap`), so dropping stow
  means a rebuild, when the deferred round trips run.
- stack's `packages/bash/install.sh` comment ("letting stow abort") is reworded in plan 02, which
  has a stack checkout.

### 5. Documentation

Core `AGENTS.md`: "Dependencies"; "Repository Layout" (`install.sh`'s paragraph, and "like stow
and yq it is base software"); "wsm: workspaces" (the stow bullet and its ignore-list sub-bullet,
`wsm add`'s "as stow's are", "one reason wsm uses stow", "remove/implode unstow"); "Source
Precedence" (`PPM_IGNORE_ARGS`, "a stow conflict"); "Claiming and Protecting Files" (`--no-folding`
in `ppm file add`, "claim's stow", `force_remove_conflicts`); "Shell Integration" (`stack/bash`:
"otherwise stow aborts"); "Key Files" (`protected.yml` "seeded into stow's ignore list"); "Package
Structure" (`install.sh` "brews it alongside stow and yq"); "Lib Structure" (`link.sh`, the
bootstrap set, `installer.sh`'s functions); "Testing" (the wsm line). The table at the top
(`packages/`: "GNU Stow plus install hooks").

READMEs: `README.md` (the table, what install.sh installs, `--skip-deps`, `anfs implode --all`),
`packages/ppm/README.md` (mise "alongside `stow` and `yq`"), `packages/wsm/README.md` (install,
"Stowed, as ppm stows packages", exit `69`). Comments: `resolve.sh` ("ppm's shared stow ignore
list"), `system.list` ("a stow symlink"), `sources.sh`, `packages.sh`'s layer comments,
`installer.sh`'s header, `implode.sh`'s.

The words "stow" and "stowed" stay where they mean "linked into place"; `ppm` help text keeps
them.

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

ppm `-n`: `ppm install -n` on a fresh, an installed and a conflicting package prints the plan and
changes nothing (the target tree, the tracker and the hooks' marker files are unchanged).

ppm: every existing suite passes. `backup.bats` drives `stow_subdir`, `unstow_package`,
`PPM_IGNORE_ARGS` and `_stow_ignore_arg` directly and skips without stow: its helpers move to the
new names, its `command -v stow` skip goes, and its cases still hold. New cases for a layered
package (the higher layer's file is skipped by exact name, including a name with `+`),
`README.md` under `home/` is linked, and an install made with stow links is taken over with no
link rewritten (compare inodes).

wsm: every existing suite passes with no `stow` on PATH. `install.bats`' "stow links" case loses
its `.stow-global-ignore` check; "absolute links from wsm 0.3" becomes the plan's `replace`.

pmm's and pcm's suites are off (ROADMAP, 2026-10-19); this plan does not touch them.

## Verification

Every command runs in the job: `anfs work run <job> core -- <cmd> </dev/null`. Nothing that
boots a VM or a box runs.

- [ ] While working, the file you touch: `bats packages/anfs/tests/link.bats`,
      `packages/ppm/tests/backup.bats`, `packages/wsm/tests/install.bats`
- [ ] `bats packages/{anfs,ppm,wsm}/tests`, with `stow` removed from PATH: the proof that
      nothing needs it
- [ ] anfs-dev: `anfs work run <job> anfs-dev -- bats packages/dev/tests/work.bats`, stow off PATH
- [ ] Once, at the end: `packages/anfs/tests/all`
- [ ] On an isolated copy of this Mac's `$HOME` state (47 packages, 228 tracked files on
      2026-10-21): `ppm install -n` for every installed package plans only `keep`
- Deferred to the end of the roadmap (named in `log.md`, not run): `roundtrip debian` from a
  bootstrap snapshot with no stow, `$HOME` diff clean; `roundtrip-macos`
