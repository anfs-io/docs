---
plan: wsm-spaces/02-repo-lib
job: wsm-02
date: 2026-10-21
---

# Log: wsm-spaces/02-repo-lib

## What was built

- **`lib/anfs/repo.sh`** (new; bash floor, not the bootstrap set). Functions only, no messages of
  its own (git speaks on error):
  - `anfs_repo_name <url>` prints the directory git clone would make (`app` for
    `git@host:org/app.git`, `https://host/org/app/`, `/srv/app.git`, `/srv/app/.git`, `host:app`);
    nothing when none. `anfs_repo_name_reply` is the same into `REPLY`, no fork (wsm's entry loop
    and `gitsrc_parse_line` use it).
  - `anfs_repo_clone <url> <dir> [ref]`: `git clone -q --`, then `git checkout -q <ref> --` (branch,
    tag, SHA). 2 when `<dir>` exists (a dangling link too), untouched; 1 when the clone or the
    checkout fails, and `<dir>` is removed. Parents are made.
  - `anfs_repo_status`, `anfs_repo_ahead` (moved from `gitsrc_dir_status`/`gitsrc_dir_ahead`,
    unchanged states), `anfs_repo_dirty` (`status --porcelain` non-empty).
  - `anfs_repo_update <dir>`: 2 when dirty, else `git pull -q` (1 on failure).
  - `anfs_repo_ssh_url <url>` (GitHub https to ssh, others unchanged), `anfs_repo_ssh <dir>`
    (switches origin, prints the new URL; 1 when there is nothing to switch).
- **`sources.sh` on it.** It sources `repo.sh` from the directory it really lives in (`BASH_SOURCE`,
  each link resolved with `readlink`, one fork when it is a link), so every tool that sources
  `sources.sh` has the library with no change, and a pull that brings `repo.sh` works before
  `core/anfs` is installed again. `gitsrc_parse_line` and `gitsrc_add` name a source with
  `anfs_repo_name` (`basename` gone: a trailing `/` gives the name, not an empty alias).
  `gitsrc_update` clones with `anfs_repo_clone`, skips with `anfs_repo_dirty`, pulls with
  `anfs_repo_update`; marks, TTL, `--auto`'s summary and the local-path link unchanged.
  `gitsrc_ssh` keeps the list rewrite, switches the remote with `anfs_repo_ssh`.
  `gitsrc_dir_status`, `gitsrc_dir_ahead` and `gitsrc_github_ssh_url` are gone (no caller left in
  core, anfs-dev or stack; checked read-only in the live sources).
- **wsm** (`WSM_VERSION` 0.6.0): `_repo_name` gone (`_repo_entries` calls `anfs_repo_name_reply`);
  `_space_repositories` clones with `anfs_repo_clone`, with a comment that wsm never calls
  `anfs_repo_update`; `repo ls` reads `anfs_repo_status`/`anfs_repo_ahead`. New **`wsm repo ssh
  [SPEC]`**: each declared repository of a space under SPEC (invalid ones skipped) whose
  `space.yml` URL is GitHub https is noted (`<id>: https in space.yml`); each cloned one of an
  installed space has its GitHub https origin switched (`<id>: origin -> git@...`), otherwise
  `<id>: already SSH`. A SPEC naming no space is 1. `space.yml` is never edited. Help epilogue: a
  repository is never pulled or deleted; `repo ssh`.
- **Docs**: core AGENTS.md ("wsm: spaces": cloned with `anfs_repo_clone`, never `anfs_repo_update`,
  `repo ssh`, why no `update`/`add`/`remove`; "Lists and --json": a clone's state is
  `anfs_repo_status`; "Lib Structure": `repo.sh`, and `sources.sh` loads it; "Testing"),
  `packages/wsm/README.md` (`repo ssh`, the alignment with `anfs src`), `packages/anfs/README.md`.

`grep -rn 'git clone\|git -C .* pull' packages/anfs packages/wsm` (tests aside) finds only
`repo.sh` (and comments that say "git clone would make").

## Commits (branch wsm-02, not pushed)

core:
- `5a13b1e` lib/anfs: repo.sh, one clone at one path (name, clone [ref], status, ahead, dirty, update, ssh); sources.sh on it, loaded from where sources.sh really lives
- `b02c5e4` wsm: repo clones and reads state through lib/anfs/repo.sh (never a pull); repo ssh [SPEC]; 0.6.0
- `9e8cb73` docs: repo.sh in AGENTS.md (wsm: spaces, Lists and --json, Lib Structure, Testing) and the anfs README

The pre-commit hook bumped nothing ("packages/anfs — already 0.2.14 since 0.2.12 was pushed",
"packages/wsm — already 0.1.8"). `WSM_VERSION` was set to 0.6.0 by hand (a new command).

## Decisions made during the work

- **`anfs_repo_name_reply`** beside `anfs_repo_name`, as `gitsrc_dir`/`gitsrc_dir_reply`: the
  plan names only the printing one; callers in loops need no fork.
- **A list line whose URL names no directory and gives no alias is no source** (`gitsrc_parse_line`
  returns 1; `src add` of such a URL without an alias is 64). Before, `basename` always gave
  something (`/` for `file:///`).
- **A failed checkout of the ref now leaves nothing** (the plan's contract). wsm used to keep the
  clone and warn "cloned … but could not check out"; it now warns `clone failed: <url> at '<ref>'`.
- **`anfs src update` is quieter**: clone and pull are `-q` (the library prints nothing but git's
  errors), so git's progress and "Already up to date." no longer appear; `Cloning:`/`Updating:`
  lines stay.
- **The dirty check** is one `git status --porcelain` (was `diff --quiet`, `diff --cached --quiet`
  and `status --porcelain`; the last covered the other two).
- **`repo ssh` says `already SSH`** for a clone whose origin is not GitHub https (a `file://` one
  too), as `anfs src ssh` does; it says nothing for a repository that is not cloned and not https
  in `space.yml`. SPEC is a space spec only (not a repository id, unlike `repo install`).
- **`bash.bats` had no parse list of `lib/anfs/*.sh`** (only the bootstrap set's, under the older
  bash, which `repo.sh` is not in). Added a test that every `lib/anfs/*.sh` parses under the floor
  bash, `repo.sh` among them.

## Tests (all through `anfs work run wsm-02 core …`, `</dev/null`)

- While working: `repo.bats` 7 (new: names of every form, trailing `/`, `.git`, none; clone with
  the default branch, a branch, a tag, a SHA; a bad ref and a bad URL leave nothing; an existing
  dir and a dangling link are 2; status in every state, ahead, dirty; update clean (pulled, no
  output), dirty (2, untouched), no upstream (1); `ssh_url`; `ssh` switched, ssh left, another host
  left, no clone), `sources.bats` 18 (new: an alias from a URL with a trailing `/`, a URL naming
  nothing; `sources.sh` reached through a link loads `repo.sh`), `bash.bats` 10 (new: the parse
  test), wsm 70 (new in `manage.bats`: `repo ssh` switches an https origin, leaves a `file://` one,
  notes a `space.yml` https URL without cloning or editing it, narrows by SPEC, 1 for an unknown
  SPEC; the clone cases unchanged).
- `bats packages/anfs/tests` 278, `bats packages/wsm/tests` 70, `bats packages/ppm/tests` 50,
  `bats packages/psm/tests` 12: all pass.
- **Once, at the end**: `packages/anfs/tests/all`: **410 passed, 0 failed**, 28 files (pmm and pcm
  off), 42 s.

## Not run

- For the person after merge: `anfs src update` and `anfs src list` on this Mac, `wsm repo ls`,
  both as before (`src update` without git's own progress lines now); `wsm repo ssh`.
- Deferred to the end of the roadmap (boot a box or a VM): `roundtrip debian`, `roundtrip-macos`.
- pmm's and pcm's suites are off (ROADMAP, 2026-10-19). pmm, pcm and its provisioner source
  `sources.sh` unchanged; their `sources.bats`/helpers were not run.
- Nothing was installed on this host; no manual run touched the real HOME or sources.

## Open questions

- **A `sources.sh` that is a copy, not a link** (e.g. after `ppm file protect` of it) would look for
  `repo.sh` beside the copy and fail to load. Linked files (ppm's default) are fine. Acceptable, or
  should `sources.sh` fall back to `$ANFS_SOURCES_HOME/core/…/repo.sh` as wsm and ppm do for
  `link.sh`?
- **`anfs src update`'s quieter output** (above): fine, or should the pull's summary stay?

## By hand, for the person after merge

1. `anfs src update` (pulls core: `sources.sh` finds the new `repo.sh` through its link before any
   install), then `ppm install core/anfs core/wsm` (links `repo.sh`, `wsm` 0.6.0).
2. `anfs src list`, `wsm repo ls`: as before. `wsm repo ssh` to switch any https clones.
