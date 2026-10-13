---
---

# Plan 02 — `~/work`: a directory per job, and `ppm work`

## Context — read these files first

- `units/dev-source.md` — the worktree sketch; plan 01's `log.md` (the `dev` package is in
  `anfs-dev` by now)
- `anfs-dev` `packages/dev/home/.local/lib/ppm/{hooks,container,vm}.sh` — how the package adds
  commands to ppm (`cli_group`, `cli_cmd`, `cmd_*`), and how `hooks.sh` finds the sources' repos
- `anfs-dev` `packages/dev/tests/{harness,roundtrip,roundtrip-macos}` — `ROUNDTRIP_SRC_ROOT`
- core `packages/pmm/tests/{helper.bash,e2e}` — `PMM_SOCK_DIR`, `PMM_SSH_PORT_BASE`, the temp HOME
- core `packages/pcm/tests/helper.bash` — what a pcm test shares with the host (podman)
- `ROADMAP.md`, "For an agent running this list"
- `machine-images/01-fold-pim-into-pmm/log.md` — the live machine's sockets deleted by a test
  run: the accident this plan's isolation rules come from

## Overview

Work on the toolkit is done in branch worktrees, so the live install (the checkouts under
`~/.local/share/anfs/sources/`, which the tools on PATH are linked from) stays on main. So far
the worktrees were made by hand under `~/anfs-dev`, one job at a time.

After this plan a job is a directory under `~/work` holding a worktree of every repo it touches,
all on one branch, made and removed by one command. Two jobs can run their tests at the same
time without meeting each other or the live install.

## Implementation

### 1. The layout

```
~/work/                       $ANFS_WORK_HOME (anfs.conf), default ~/work
  <job>/                      the job's name is the branch's name
    core/  stack/  anfs-dev/  worktrees of the clones under sources/, named by alias
    docs/                     a worktree of a repository a space clones
    .job.yml                  what the job is: branch, repos, its number, created
```

### 2. Commands (`ppm work`, in the `dev` package)

```
ppm work new <job> [repo...] [--from REF]    worktrees on a new branch <job>, from REF (default: each repo's main)
ppm work add <job> <repo...>                 another repo joins the job
ppm work list                                JOB BRANCH REPOS STATE (clean, dirty, ahead N, merged)
ppm work path <job> [repo]                   and `ppm work cd`, through the shell wrapper ppm cd uses
ppm work env <job>                           the variables below, as `export` lines
ppm work run <job> -- <cmd...>               run a command with them set, in the job's directory
ppm work remove [-f] <job>                   worktrees and branches; refuses unmerged or dirty without -f
```

- A `repo` is a source's alias (`core`, `stack`, `anfs-dev`, `user`), or the name of a repository
  some installed space's `space.yml` clones (`docs`). With none given, `new` takes `core`.
- A source that is a link to a local path, or has no git, is refused.
- `new` runs `ppm hooks install` on each worktree's repo (hooks are per repo, so this is a no-op
  after the first job) and never pushes.
- `remove` runs `git worktree remove` and deletes the branch when it is merged into the repo's
  main; with `-f` it deletes an unmerged branch after `anfs_confirm` naming the commits that go.
- `list` goes through `anfs_table`; keys `job`, `branch`, `repos[]`, `state`, `number`.

### 3. Isolation: what a job's tests may share

Each job gets a number at `new` (the lowest free one, from 1, kept in `.job.yml`). `ppm work env`
derives from it, and the test helpers and e2e scripts use these when they are set:

| Variable | Value | Why |
| --- | --- | --- |
| `ANFS_WORK_JOB` | the job's name | so a helper can tell it runs inside a job |
| `ROUNDTRIP_SRC_ROOT` | `~/work/<job>` | the round trips test this job's trees |
| `PMM_SOCK_DIR` | `/tmp/pmm-work-<uid>-<n>` | never the live `/tmp/pmm-<uid>` |
| `PMM_SSH_PORT_BASE` | `2222 + 100 × n` | two jobs' machines never want the same port |
| `TMPDIR` | `/tmp/anfs-work-<uid>-<n>` | temp homes are found and cleaned per job |

- The round trips take a source the job does not hold from the installed sources, by alias.
- Shared, and read-only to every test: the ISO store (`~/.cache/anfs/isos`). A test that would
  write there fails instead (the helpers set `ANFS_ISO_HOME` to a temp dir unless the test is an
  e2e, which only reads).
- Shared, with a limit: tart allows two macOS guests in all, and the live install may be using
  one. `roundtrip-macos` and the tart tests take a host-wide lock
  (`$ANFS_STATE_HOME/work/tart.lock`) and say who holds it.
- Shared: podman. The harness already names its containers; add the job's number to the name so
  two round trips do not collide.
- **No test in any suite may name `/tmp/pmm-$(id -u)`, a real `*_HOME`, or use a broad `pkill`.**
  A bats test in core's anfs suite greps the tests for those and fails on a match.

### 4. The agent section

`ROADMAP.md`'s "For an agent running this list" is rewritten around this: an orchestrator runs
`ppm work new <job> <repos>` and gives each agent the job's path; agents run tests through
`ppm work run <job> --`; the list of things never to touch is the same and now enforced by
section 3 where it can be.

### 5. Leaving `~/anfs-dev`

Not for an agent. In the log, for the person: once `machine-images` is merged, `git worktree
remove` the three checkouts under `~/anfs-dev` and delete the directory. Nothing reads the old
default after this plan.

### 6. Documentation

`anfs-dev`'s CLAUDE.md: a "Working on the toolkit" section (the layout, the commands, the
isolation table). core's CLAUDE.md "Testing": two lines and a pointer.

## Test Spec

`anfs-dev` `packages/dev/tests/work.bats` (local bare repos as sources, as the anfs suites do):

- `new` with two repos: two worktrees on branch `<job>`, `.job.yml` written, number 1; a second
  job gets 2; after removing the first, a third job gets 1.
- A repository of a space (`docs`) is found through `space.yml` and gets a worktree.
- `env`: the five variables, and two jobs' values differ.
- `list` states: clean, dirty, ahead, merged; `--json` keys.
- `remove`: refuses dirty and unmerged; removes a merged job's worktrees and branches; `-f`
  asks, then removes.
- `add` to an existing job; a local-path source is refused; a job name that is not a valid
  branch name is refused (64).

core: the grep test of section 3; `packages/pmm/tests/helper.bash` and `e2e` honour
`PMM_SOCK_DIR` and `PMM_SSH_PORT_BASE` from the environment when set.

## Verification

- [ ] `bats packages/dev/tests </dev/null` in `anfs-dev`; `bats packages/{anfs,pmm}/tests </dev/null` in core
- [ ] Two jobs at once on this Mac: `ppm work run a -- packages/pmm/tests/e2e debian` and the
      same for `b`, started together; both pass, and `/tmp/pmm-<uid>` is untouched
- [ ] `ppm work run <job> -- packages/dev/tests/roundtrip debian` tests the job's trees (a
      marker file in the job's core shows up in the box)
- [ ] `ppm work remove` of a merged job leaves `git worktree list` clean in every repo

## Open Questions

1. **`ppm work` or `anfs work`?** The `dev` package adds commands to ppm today (`ppm container`,
   `ppm vm`, `ppm hooks`), so `ppm work` is consistent. It spans sources, which is anfs's kind
   of thing, but anfs has no extension seam. Recommended: `ppm work` now.
2. **Does a job get its own install to try by hand** (`ppm work shell <job>`: a shell with a
   temp HOME whose tools are the job's)? The e2e scripts already build one. Recommended: yes,
   as a follow-up; it is what makes "try my branch" one command.
