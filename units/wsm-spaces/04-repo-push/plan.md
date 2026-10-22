---
---

# Plan 04 — `push`: `anfs src push` and `wsm repo push` on `lib/anfs/repo.sh`

Written 2026-10-22 against main (which holds plans 01–03 of this unit). Decided by the person on
2026-10-22: push many repos without cd'ing into each one.

## Context — read these files first

The job is `anfs work new repo-push core docs`. Paths are core's.

- `packages/anfs/home/.local/lib/anfs/repo.sh` — `anfs_repo_ahead`, `anfs_repo_dirty`,
  `anfs_repo_status`, `anfs_repo_update` (the models); `tests/repo.bats`
- `packages/anfs/home/.local/lib/anfs/sources.sh` — `gitsrc_collect`, `gitsrc_dir_reply`,
  `gitsrc_update` (what `anfs src update` calls)
- `packages/anfs/home/.local/bin/anfs` — the `src` group, `complete_source`
- `packages/wsm/home/.local/bin/wsm` — `_repo_entries`, `_repo_invalid`, `cmd_repo_list`,
  `cmd_repo_install` (`_repo_split`), `cmd_repo_ssh`, `complete_space`
- `packages/anfs/home/.local/lib/anfs/table.sh` (`anfs_table`, `_anfs_tone`), `cli.sh`
  (`cli_usage_err`, `CLI_OPT`), `log.sh` (`anfs_warn`, `anfs_fail`)
- `packages/anfs/tests/registry.bats` — every `anfs_table` header held to the list shape
- `AGENTS.md` — "Lists and --json", "The Command Line", "wsm: spaces", "Lib Structure"
- plan 02's `plan.md` and `log.md` (how `repo.sh` is shared: `sources.sh` loads it, wsm sources it)

## Overview

`anfs src push` and `wsm repo push` push the current branch of each named repo (or of every repo
ahead of its upstream, `--all`) with plain `git push`, and print one row per repo in the toolkit's
list shape. The git half is one library function, `anfs_repo_push`; the command half (names,
`--all`, `-n`, the rows, the exit status) is one helper both tools call, `anfs_repo_push_cli`.
Each tool only says which repos it has and where their clones are.

## Implementation

### 1. The library: `anfs_repo_push <dir> [-n]`

In `repo.sh`, beside `anfs_repo_update`. Prints nothing.

- Not a clone (no `.git`): 5. A detached HEAD: 4. A branch tracking nothing: 3. Nothing ahead of
  the upstream (as of the last fetch; nothing is fetched): 2.
- Else, with `-n`, 0 without pushing. Without it, `git push -q` with no arguments: never
  `--force`, never `--set-upstream`, tags only as the repo's config says (`push.followTags`).
  0 pushed; 1 when git fails (non-fast-forward, auth, unreachable), with git's first error line in
  `REPLY` (a ` ! [rejected] ...` ref line, `error:` or `fatal:`; whitespace squeezed).
- `ANFS_REPO_BRANCH`, `ANFS_REPO_UPSTREAM` and `ANFS_REPO_AHEAD` hold what was found (empty when
  not), for the caller's row.

### 2. The CLI helper: `anfs_repo_push_cli <kind> <list command> [name...]`

Also in `repo.sh` (a second section; its header says this one prints, and needs `cli.sh`,
`table.sh` and `log.sh`, which every caller has). The caller fills `ANFS_PUSH_IDS` (every repo,
in its list's order, named as its list names it) and `ANFS_PUSH_DIRS` (each one's clone; empty
when it has none), then calls it with its kind (`source`, `repository`), its list command (for
messages) and the names typed. Options come from `CLI_OPT` (`all`, `dry-run`).

- Names: each must be one of `ANFS_PUSH_IDS`, else `no <kind> '<name>' (<list command>)`, exit 1,
  before anything is pushed. A name given twice is pushed once.
- `--all`: every repo ahead of its upstream (`anfs_repo_ahead` > 0), in list order. None: `Nothing
  to push: no <kind> is ahead of its upstream` (`[]` with `--json`), 0.
- Neither: a usage error (64) that lists the repos ahead and suggests `--all`
  (`name the sources to push, or --all for every one ahead of its upstream: core, stack`; when
  none is: `... (none is ahead of its upstream)`). Both: a usage error (`names or --all, not both`).
- One row per repo, `<KIND> STATUS BRANCH UPSTREAM`, keys `<kind> status branch upstream commits
  error` (`commits` and `error` in `--json` only):

  | `anfs_repo_push` | STATUS | failed |
  | --- | --- | --- |
  | 0 | `pushed N`, `pushed N (uncommitted changes left)` when dirty; `-n`: `would push N` (same suffix) | no |
  | 2 | `up to date` | no |
  | 3 | `no upstream` | yes |
  | 4 | `detached` | yes |
  | 1 | `rejected`; git's line is a warning (`<tool>: <id>: rejected: <line>`) and `error` | yes |
  | 5 | `not cloned` (a source not cloned or not a git repo; a space not installed) | yes |

- Every repo is tried; exit 1 when any failed, else 0.
- `_anfs_tone` (table.sh): `rejected` red, `detached` yellow, as `dirty`/`no upstream` are.

### 3. `anfs src push [source...] [--all] [-n] [--json]`

`cmd_src_push` collects the sources (`gitsrc_collect`), each one's directory
(`gitsrc_dir_reply`: the clone, or the link to a local path), and calls the helper with `source`
and `anfs src list`. The argument's kind is `source` (completed by `complete_source`, as
`src update`). Declared after `src update`.

### 4. `wsm repo push [repository...] [--all] [-n] [--json]`

`cmd_repo_push` loads the spaces as `repo list` does; the candidates are every repository a space
declares with a valid path (`_repo_entries`, `_repo_invalid`), named `<space id>/<path>` as `repo
list` names them (the id `_repo_split` would resolve: a path inside a child space is invalid, so
the two agree), its clone the installed space's directory plus the path, empty while the space is
not installed. The help epilogue's "never pulled or deleted" sentence gains: `repo push` pushes
what is ahead. `WSM_VERSION` 0.10.0 (a new command).

Completion of `repository` (`complete_repository`, new; `repo install`'s `[spec|repository]`
completes nothing today): the clones in the installed spaces, read off the file system only (no
yq, no git, as the rule for `complete_<kind>` says): `<id>/<path>` for each `<path>/.git` up to
three levels below an installed space's directory, no part of `<path>` a link (a `links:` entry
out of the tree is not a clone of the space), each once.

### 5. Documentation

- Core `AGENTS.md`: "Lists and --json" (rows for `anfs src push` and `wsm repo push`); "wsm:
  spaces" (`repo push`); "Lib Structure" (`repo.sh`: push, and the helper).
- `packages/anfs/README.md`, `packages/wsm/README.md`: the commands.
- anfs's header comment (`src <add|remove|list|ssh|update|push>`).
- `registry.bats`: the helper's header is built from its kind, so the header check also reads
  each `anfs_repo_push_cli <kind>` call (the kind upper-cased must be a list kind) and skips the
  helper's own `anfs_table` line.

## Test Spec

Every remote is a temporary local bare repo over a path or `file://`; nothing is pushed anywhere
else.

- `repo.bats` (`anfs_repo_push`): ahead (pushed, the remote has the commit, branch/upstream/ahead
  set), up to date (2), no upstream (3), detached (4), not a clone (5), the remote moved on
  (1, `REPLY` is git's `! [rejected] ... (fetch first)` line, nothing pushed), dirty (pushed,
  the changes stay), `-n` (0, the remote unchanged), tags not pushed.
- `push.bats` (new; the real `anfs` in a temp `$HOME`, sources cloned from bare repos): `src push`
  of names (each outcome's row, exit 1 with one failure, every repo tried), `--all` against
  names (only the ahead ones), `-n` (rows, nothing pushed), `--json` keys, the no-argument
  error (64, the ahead list and `--all`), both (64), an unknown name (1, nothing pushed),
  completion of the argument.
- wsm `manage.bats`: `repo push` of a repository id (pushed), `--all` across spaces, a space not
  installed (`not cloned`), the no-argument error, an unknown id, `complete_repository` (clones
  of installed spaces, not a linked-out tree).
- `registry.bats`: passes with the helper's header.

## Verification

Every command runs in the job: `anfs work run repo-push core -- <cmd> </dev/null`. Nothing is
installed on this host, nothing is pushed to a real remote, and the new commands are not run
against the real sources or spaces.

- [ ] While working: `bats packages/anfs/tests/repo.bats`, `push.bats`, `registry.bats`,
      `bats packages/wsm/tests/manage.bats`
- [ ] Once, at the end: `packages/anfs/tests/all`
- For the person after merge (in `log.md`): reinstall core/anfs and core/wsm, then `anfs src push
  -n --all` and `wsm repo push -n --all` (read-only). The real push is the person's call.
