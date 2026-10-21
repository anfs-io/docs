---
---

# Plan 02 — `lib/anfs/repo.sh`: one clone at one path, for `anfs src` and `wsm repo`

Written 2026-10-21 against main (which holds `linker-01` and `ovm-01`), and plan 01 of this
unit (independent of plan 03)

## Context — read these files first

The job is `anfs work new <job> core`. Paths are core's.

- `units/wsm-spaces.md` — decision 9; plan 01's `log.md`
- `packages/anfs/home/.local/lib/anfs/sources.sh` — `gitsrc_parse_line` (the alias from
  `basename "$url" .git`), `gitsrc_add` (the same), `gitsrc_dir_status`, `gitsrc_dir_ahead`,
  `gitsrc_github_ssh_url`, `gitsrc_ssh` (the remote rewrite), `gitsrc_update` (clone, the dirty
  check, pull, the marks), `gitsrc_link_local`; `tests/sources.bats`
- `packages/wsm/home/.local/bin/wsm` — `_repo_name`, `_repo_entries`, `_repositories` (its
  `git clone -q --` and `checkout`), `cmd_repo_list`, `cmd_repo_install`
- `packages/anfs/home/.local/bin/anfs` — the `src` group (`cmd_src_*`)
- `packages/wsm/home/.local/bin/wsm` and `packages/psm/home/.local/bin/psm`, `ppm`, `pcm`, `pmm`,
  `anfs`, anfs-dev's `anfs-work`, stack's `ventoy`: everything that sources `sources.sh`
- `packages/anfs/tests/bash.bats` (the parse list of `lib/anfs/*.sh`)
- `AGENTS.md` — "Lists and --json" (a clone's state, `gitsrc_dir_status`), "Lib Structure"

## Overview

A repository at a path is cloned, inspected, pulled and switched to ssh by one library. `anfs
src` keeps its policy (a TTL, the marks, skipping a dirty clone, linking local paths) and `wsm
repo` keeps its own (clone what is missing, never pull, never delete). There is one `git clone`
and one URL parser in the toolkit's libraries.

## Implementation

### 1. The library

`packages/anfs/home/.local/lib/anfs/repo.sh`, functions only, the bash floor (5.2; not the
bootstrap set: `install.sh` clones core with its own `git clone` before any library exists, and
keeps it).

```
anfs_repo_name    <url>              the directory git clone would make: app for git@host:org/app.git,
                                     https://host/org/app/, /srv/app.git, host:app; nothing when none
anfs_repo_clone   <url> <dir> [ref]  clone (quiet, --), then check out ref (a branch, tag or SHA);
                                     1 on failure, and nothing is left at <dir>; 2 when <dir> exists
anfs_repo_status  <dir>              clean, dirty, unpushed, no upstream, "dirty, unpushed",
                                     "dirty, no upstream", local (not a repo), missing; no fetch
anfs_repo_ahead   <dir>              commits upstream lacks; nothing for no upstream
anfs_repo_dirty   <dir>              true when it has uncommitted or untracked changes
anfs_repo_update  <dir>              pull; 2 without pulling when dirty, 1 when the pull fails
anfs_repo_ssh_url <url>              https://github.com/x -> git@github.com:x; other URLs unchanged
anfs_repo_ssh     <dir>              origin from GitHub https to ssh; prints the new URL; 1 when
                                     there was nothing to switch
```

Messages are the caller's: the library prints nothing but what git prints on error.

### 2. `sources.sh` on it

- `gitsrc_parse_line` and `gitsrc_add` name a source with `anfs_repo_name` (a trailing `/` no
  longer gives an empty alias; `.git` is dropped as before).
- `gitsrc_update`: `anfs_repo_clone` for a missing source, `anfs_repo_dirty` for the skip,
  `anfs_repo_update` for the pull. The marks, the TTL, `--auto`'s summary and the local-path
  link stay here, unchanged.
- `gitsrc_ssh`: the list rewrite stays; the remote is `anfs_repo_ssh`; `gitsrc_github_ssh_url`
  is `anfs_repo_ssh_url`.
- `gitsrc_dir_status` and `gitsrc_dir_ahead` go; `gitsrc_status` and `gitsrc_list_file` call
  `anfs_repo_status`/`anfs_repo_ahead`.
- `sources.sh` sources `repo.sh` from the directory it really lives in (its own link resolved),
  so every tool that sources `sources.sh` has it with no change, and a pull that brings `repo.sh`
  works before `core/anfs` is installed again (the same problem `link.sh` met in linker 01).
- `gitsrc_is_git_url` stays in `sources.sh` (a source-list question; onboard, ppm, pmm and
  `anfs-work` call it).

### 3. `wsm repo` on it

- `_repositories` clones with `anfs_repo_clone`; `_repo_name` goes (`_repo_entries` calls
  `anfs_repo_name`). `wsm repo list` reads `anfs_repo_status`/`anfs_repo_ahead`.
- Never a pull: `wsm` does not call `anfs_repo_update` (a comment says why: what is checked out is
  the user's).
- **`wsm repo ssh [SPEC]`** (cheap, as `anfs src ssh`): each clone under SPEC whose origin is a
  GitHub https URL is switched to ssh (`<id>: origin -> git@...`). `space.yml` is the source's and
  is not edited; a repository it declares by https is noted (`<id>: https in space.yml`).
- `wsm repo` aligned with `anfs src`, and why not further: `list` (the same columns' meaning:
  `REPOSITORY STATUS URL`, `ahead`), `install` (`src install` installs a source's resources;
  `repo install` clones), `ssh`. No `update` (decision 9: space repositories are never pulled),
  no `add`/`remove` (a repository is declared in the source's `space.yml`; wsm does not edit it).
  `-n` on `repo ssh` is linker plan 04's.

### 4. Documentation

Core `AGENTS.md`: "Lists and --json" (a clone's state is `anfs_repo_status`); "Lib Structure"
(`repo.sh`, and that `sources.sh` loads it); "wsm: spaces" (`repo ssh`, never pulled).
`packages/wsm/README.md` (`repo ssh`). `wsm help` (the group's commands).

## Test Spec

- `packages/anfs/tests/repo.bats` (temp dirs, bare repos over `file://`): `anfs_repo_name` on the
  URL forms above, a trailing `/`, `.git`, none; `anfs_repo_clone` with a branch, a tag, a SHA, a
  bad ref (nothing left), a bad URL (nothing left), an existing dir (2); `anfs_repo_status` in
  each state; `anfs_repo_update` clean (pulled), dirty (2, untouched), no upstream (1);
  `anfs_repo_ssh_url` and `anfs_repo_ssh` (https switched, ssh left, a non-GitHub remote left).
- `sources.bats`: unchanged and passing; an alias from a URL with a trailing `/`.
- wsm: the clone cases unchanged and passing; `repo ssh` switches a clone's https origin and
  notes the `space.yml` URL; `space.yml` is unchanged after.
- `bash.bats`: `repo.sh` in the parse list.
- `grep -rn 'git clone\|git -C .* pull' packages/anfs packages/wsm` finds only `repo.sh` and the
  tests.

## Verification

Every command runs in the job: `anfs work run <job> core -- <cmd> </dev/null`. Nothing that boots
a VM or a box runs, and nothing is installed on this host.

- [ ] While working, the file you touch: `bats packages/anfs/tests/repo.bats`,
      `bats packages/anfs/tests/sources.bats`, `bats packages/wsm/tests/<file>`
- [ ] `bats packages/anfs/tests`, `bats packages/wsm/tests`, `bats packages/ppm/tests`,
      `bats packages/psm/tests` (they source `sources.sh`)
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`): `anfs src update` and `anfs src list` on this
  Mac, `wsm repo ls`, both as before.
- Deferred to the end of the roadmap (named in `log.md`): `roundtrip debian`, `roundtrip-macos`.
