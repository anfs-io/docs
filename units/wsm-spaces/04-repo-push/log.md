---
plan: wsm-spaces/04-repo-push
job: repo-push
date: 2026-10-22
---

# Log: wsm-spaces/04-repo-push

## What was built

- **`anfs_repo_push <dir> [-n]`** (`lib/anfs/repo.sh`, beside `anfs_repo_update`; prints
  nothing). Plain `git push -q` of the current branch: no force, no `--set-upstream`, tags only as
  the repo's config says. Exit status is the outcome: 0 pushed (with `-n`, would be), 1 git failed
  (git's first error line in `REPLY`: a ` ! [rejected] ...` ref line, `error:`, `fatal:`,
  whitespace squeezed), 2 nothing ahead, 3 no upstream, 4 detached, 5 not a clone. Sets
  `ANFS_REPO_BRANCH`, `ANFS_REPO_UPSTREAM`, `ANFS_REPO_AHEAD`. No fetch: "ahead" is as of the last
  fetch, so a remote that moved on shows as ahead and is then `rejected`.
- **`anfs_repo_push_cli <kind> <list command> [name...]`** (`repo.sh`, its own section; the
  header now says this one prints and needs `cli.sh`, `table.sh`, `log.sh`). The caller fills
  `ANFS_PUSH_IDS`/`ANFS_PUSH_DIRS`; the helper resolves names (all before any push; unknown: `no
  <kind> '<name>' (<list command>)`, 1; a repeat pushed once), `--all` (every one ahead, list
  order; none: `Nothing to push: no <kind> is ahead of its upstream`, `[]` with `--json`), neither
  (64: `name the sources to push, or --all for every one ahead of its upstream: a, c`, or `(none
  is ahead of its upstream)`), both (64: `names or --all, not both`). One row each, `<KIND> STATUS
  BRANCH UPSTREAM`, keys `<kind> status branch upstream commits error`. Statuses: `pushed N`,
  `pushed N (uncommitted changes left)`, `would push N` (`-n`, same suffix), `up to date`, `no
  upstream`, `detached`, `rejected` (git's line also a warning, `anfs: <id>: rejected: <line>`),
  `not cloned`. Every repo is tried; exit 1 when any was not pushed (`up to date` is not a
  failure).
- **`anfs src push [source...] [--all] [-n] [--json]`**: every source in `gitsrc_collect` order,
  its directory from `gitsrc_dir_reply`; argument kind `source` (`complete_source`, as `src
  update`).
- **`wsm repo push [repository...] [--all] [-n] [--json]`** (`WSM_VERSION` 0.10.0): every
  repository a space declares with a valid path, `<space id>/<path>` as `repo ls` names it, its
  clone the installed space's directory plus the path (none while not installed: `not cloned`).
  New `complete_repository`. Help epilogue: `repo push pushes the ones you name, or every one
  ahead of its upstream (--all)`.
- `_anfs_tone` (table.sh): `rejected` red, `detached` yellow.
- **Docs**: core AGENTS.md ("Lists and --json": a row for both push commands and the tones; "wsm:
  spaces"; "Lib Structure": `anfs_repo_push`, `anfs_repo_push_cli`), `packages/anfs/README.md`,
  `packages/wsm/README.md`, anfs's header comment.

## Commits (branch repo-push, not pushed)

core:
- `5ada1ed` lib/anfs/repo.sh: anfs_repo_push and anfs_repo_push_cli; anfs src push; rejected/detached toned. Tests: repo.bats, push.bats, registry.bats reads the helper's kind
- `55fb323` wsm repo push [repository...] [--all] [-n]; complete_repository; 0.10.0
- `de7afbb` docs: push in AGENTS.md and the anfs and wsm READMEs
- `2cbc7f9` repo.sh: anfs_repo_push reads ANFS_REPO_AHEAD by name (settings.bats)

The pre-commit hook bumped `packages/wsm` to 0.1.10; `packages/anfs` was already 0.2.16 since
0.2.15 was pushed. `WSM_VERSION` 0.10.0 by hand (a new command).

docs: the plan (`c12dd23`, with its row in `units/wsm-spaces.md`), this log.

## Decisions made during the work

- **Names and `--all` together is a usage error** (the decision named only "neither").
- **Rows in the order asked**: names as typed, `--all` in the list's order.
- **Not a clone is one word, `not cloned`**, for both tools (a source not cloned or not a git
  repo, a space not installed), and a failure when named. `--all` never meets it (only clones can
  be ahead), nor `no upstream`/`detached`.
- **git's error line goes to stderr as a warning and to `--json`'s `error`**, not into a column: the
  table stays `<KIND> STATUS BRANCH UPSTREAM`. The preferred line is git's ` ! [rejected] main ->
  main (fetch first)` (it comes before `error: failed to push some refs`, which says less).
- **`-n` exits as the push would for what it can know** (no upstream, detached, not cloned are 1);
  it cannot know a rejection (nothing is contacted).
- **The table is printed after every push** (it is aligned over all rows); rejections are warned as
  they happen. git's own prompts (ssh passphrase, credentials) reach the terminal, unchanged.
- **`complete_repository` is new**: `repo install`'s `[spec|repository]` completes nothing today
  (no `complete_spec|repository`), so there was no repository completion to share. It reads the
  file system only (`<path>/.git` up to three levels under each installed space, no part of the
  path a link, so a `links:` entry out of the tree, e.g. `sources`, is not offered), no yq or git.
  A clone deeper than three levels is not completed but still resolves.
- **registry.bats**: the helper's `anfs_table` header is built from its kind, so the header check
  skips that one line and checks each `anfs_repo_push_cli <kind>` call's kind instead.

## Tests (all through `anfs work run repo-push core …`, `</dev/null`)

Every remote is a temporary bare repo under the test's directory; nothing was pushed anywhere else,
and neither command was run against the real sources or spaces.

- `repo.bats` 10 (3 new): ahead pushed (branch/upstream/ahead set, tags not pushed), nothing ahead
  2; `-n` pushes nothing; dirty pushes and keeps the change; no upstream 3 (no branch made on the
  remote), detached 4, not a clone 5, the remote moved on 1 with `! [rejected] main -> main (fetch
  first)` and the remote unchanged, an unreachable remote 1 with a `fatal:` line.
- `push.bats` 8 (new; the real `anfs` in a temp `$HOME`): names with every outcome in one run (exit
  1, every source tried, only the right ones pushed), all pushed/up to date is 0; `--all` (none:
  the message and `[]`; then only the ahead ones, in list order); neither (64 with the ahead list,
  or "none is ahead"), both (64); `-n`; an unknown name (1, nothing pushed), a repeated name;
  a listed source not cloned; `--json` keys; completion.
- `manage.bats` (wsm) 34 (2 new): `repo push` names, `-n`, `--all`, nothing ahead, a space not
  installed, an unknown id, `--json`, the no-argument error; `__complete repo push` offers the
  clone and not a git repo under a `links:` tree out of the space.
- `registry.bats` 43, `table.bats` 14, `bash.bats` 10, `settings.bats`: pass.
- **The full run**: `packages/anfs/tests/all` first gave 661 passed, 1 failed: `settings.bats`
  read `${ANFS_REPO_AHEAD:-0}` as an undocumented setting. Fixed (`2cbc7f9`), then run again:
  **662 passed, 0 failed**, 48 files (pmm and pcm off), 88 s.

## Not run

- pmm's and pcm's suites are off (ROADMAP, 2026-10-19); this plan does not touch them.
- Nothing boots a VM or a box; nothing was installed on this host.

## Open questions

- None blocking. If the person wants git's rejection line in the table itself (a `NOTE`-like
  column), it is one key away: it is already `error` in `--json`.

## By hand, for the person after merge

1. Reinstall what the job touched: `ppm install core/anfs core/wsm` (`lib/anfs/repo.sh`,
   `table.sh` and `anfs` are core/anfs's; `wsm` 0.10.0 is core/wsm's).
2. A read-only check: `anfs src push -n --all` and `wsm repo push -n --all` (what would be
   pushed: the branch, upstream and commit count; nothing is pushed).
3. The real push (`anfs src push --all`, `wsm repo push --all`, or by name) is your call.
