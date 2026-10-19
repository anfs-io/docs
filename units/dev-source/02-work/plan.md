---
---

# Plan 02 (runs first) — `~/work`, `anfs work`, commands from packages, and the tools as `anfs <resource>`

## Context — read these files first

- `units/dev-source.md` — the worktree sketch. **This plan runs before the rest of plan 01**.
  The `dev` package itself is already in the `anfs-dev` source (moved by hand 2026-10-17); the
  test boxes, round trips and fixture are still in core
- `anfs-dev` `packages/dev/home/.local/lib/ppm/{hooks,container,vm}.sh` — how the package adds
  commands to ppm (`cli_group`, `cli_cmd`, `cmd_*`), and how `hooks.sh` finds the sources' repos
- core `packages/anfs/tests/{harness,roundtrip,roundtrip-macos}` — `ROUNDTRIP_SRC_ROOT`
- core `packages/pmm/tests/{helper.bash,e2e}` — `PMM_SOCK_DIR`, `PMM_SSH_PORT_BASE`, the temp HOME
- core `packages/pcm/tests/helper.bash` — what a pcm test shares with the host (podman)
- `ROADMAP.md`, "For an agent running this list"
- `machine-images/01-fold-pim-into-pmm/log.md` — the live machine's sockets deleted by a test
  run: the accident this plan's isolation rules come from

## Overview

Work on the toolkit is done in branch worktrees, so the live install (the checkouts under
`~/.local/share/anfs/sources/`, which the tools on PATH are linked from) stays on main. So far
the worktrees were made by hand, one job at a time. This plan's own job is the last: core and
anfs-dev worktrees under `~/work/anfs-work/`, branch `anfs-work`.

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

### 2. Commands from packages (anfs only)

ppm sources every file in `~/.local/lib/ppm/`, and a `cmd_` function there becomes a ppm
command. That is from when ppm was the only tool; the `dev` package is the one thing that uses
it for commands. anfs gets a smaller seam and ppm's stops being used for commands:

- `anfs <name>`, when anfs declares no such command, runs the executable `anfs-<name>` found in
  `~/.local/bin` with the rest of the line. Nothing is sourced into anfs, so a package cannot
  redefine anything of anfs's.
- Such an executable is a `cli.sh` tool like any other (`ventoy` is the model) and sets
  `CLI_TOOL="anfs <name>"`, so its messages and usage lines read `anfs work: ...`.
- `anfs help` lists them by name under "From packages"; `anfs help <name>` and
  `anfs <name> --help` are the executable's own. `anfs __complete <name> ...` asks
  `anfs-<name> __complete ...`; the names themselves complete from the file names (no process).
- `-d`, `--trace` and `--json` given to anfs are passed on.
- anfs's own commands win: a package cannot take a name anfs declares. `bash.bats`' list of
  entry points covers `anfs-*` in the sources.
- The other tools get no such seam. ppm keeps what is about packages: hook helpers a package
  stows into `lib/ppm` (`stack/ruby`'s `install_gem`) and `ppm_resource_<key>` handlers.

`ppm hooks` becomes `anfs hooks` this way (`anfs-hooks`; it acts on the sources' repos, not on
packages), with the same sub-commands. `hooks.sh` leaves `lib/ppm`.

### 2b. The tools as `anfs <resource>`

anfs declares one command per tool, named for what it manages, singular, with the tool's own
name as an alias:

| Command | Alias | Runs |
| --- | --- | --- |
| `anfs package` | `anfs ppm` | `ppm` |
| `anfs container` | `anfs pcm` | `pcm` |
| `anfs machine` | `anfs pmm` | `pmm` |
| `anfs skill` | `anfs psm` | `psm` |
| `anfs space` | `anfs wsm` | `wsm` |
| `anfs vault` | `anfs pvm` | `pvm` |

- Each `exec`s the tool with the rest of the line: `anfs package install git` is `ppm install
  git`, `anfs machine image build debian-13` is `pmm image build debian-13`, exit status and
  output and all. `-d`, `--trace` and `--json` given to anfs are passed on.
- A tool that is not installed (pmm, after `machine-images/05`) is not an error of the line:
  `anfs machine` says `pmm is not installed (ppm install core/pmm)` and exits 69.
- `anfs help` lists them under "Tools". `anfs help machine` is `pmm help`. `anfs __complete
  machine ...` asks `pmm __complete ...`.
- None of the six names may already be an anfs command (check `anfs help` before choosing the
  declaration). Declared commands win over the package seam, so no package can ship
  `anfs-package`.
- The six links `packages/<tool>/home/.local/bin/anfs-<x>-manager` in core (a trial of this,
  15d1aaa) are deleted.

### 2a. `anfs work` (`anfs-work`, in the `dev` package)

```
anfs work new <job> [repo...] [--from REF]
anfs work add <job> <repo...>
anfs work list
anfs work show <job>
anfs work path <job> [repo]
anfs work cd <job> [repo]
anfs work env <job>
anfs work run <job> [repo] -- <cmd...>
anfs work remove [-f] [-y] <job>
```

| Command | Does |
| --- | --- |
| `new` | Makes `~/work/<job>/`, and in it a git worktree of each named repo on a new branch `<job>`, cut from REF (default: each repo's main). Gives the job its number and writes `.job.yml`. Wires the git hooks. Never pushes. With no repo: `core` |
| `add` | Another repo joins the job: a worktree on the same branch name |
| `list` | Every job: `JOB BRANCH REPOS STATE`. State is the worst of its repos: `dirty`, `ahead N`, `clean`, `merged` |
| `show` | One job: each repo with its path, branch, commits ahead of main and whether it is dirty; the job's number; the variables `env` prints |
| `path`, `cd` | The job's directory, or one repo's worktree in it. `cd` is through the shell function `sh/anfs.sh` defines, as `ppm cd` is |
| `env` | The isolation variables (section 3) as `export` lines, for `eval` or a file an agent sources |
| `run` | Runs a command with those variables set, in the job's directory. The command's exit status is `run`'s |
| `remove` | `git worktree remove` for each repo, then deletes the branch where it is merged into main, then the directory. Refuses a dirty worktree or an unmerged branch; `-f` goes ahead after `anfs_confirm` lists the commits that would be lost |

- A `repo` is a source's alias (`core`, `stack`, `anfs-dev`, `user`), or the name of a repository
  some installed space's `space.yml` clones (`docs`).
- A source that is a link to a local path, or has no git, is refused.
- A job's name must be a valid branch name with no `/` (64).
- `list` goes through `anfs_table`; keys `job`, `branch`, `repos[]`, `state`, `number`.
- Exit codes are the toolkit's: 73 a job that exists, 1 one that does not.

Trying a job in a box (a fresh Debian, Fedora or macOS, or another user on this host) is plan
03: it replaces `ppm container`, `ppm vm` and `ppm user`, and needs `pcm run`.

### 3. Isolation: what a job's tests may share

Each job gets a number at `new` (the lowest free one, from 1, kept in `.job.yml`). `anfs work env`
and `run` derive from it, and the test helpers and e2e scripts use these when they are set:

| Variable | Value | Why |
| --- | --- | --- |
| `ANFS_WORK_JOB` | the job's name | so a helper can tell it runs inside a job |
| `ROUNDTRIP_SRC_ROOT` | `~/work/<job>` | the round trips test this job's trees |
| `PMM_SOCK_DIR` | `/tmp/pmm-work-<uid>-<n>` | never the live `/tmp/pmm-<uid>` |
| `PMM_SSH_PORT_BASE` | `2222 + 100 × n` | two jobs' machines never want the same port |
| `TMPDIR` | `/tmp/anfs-work-<uid>-<n>` | temp homes are found and cleaned per job |

- The round trips take a source the job does not hold from the installed sources, by alias.
- Shared, and read-only to every test (decided 2026-10-13: an ISO downloads once): the ISO
  store (`~/.cache/anfs/isos`). A test that would
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
`anfs work new <job> <repos>` and gives each agent the job's path; agents run tests through
`anfs work run <job> --`; the list of things never to touch is the same and now enforced by
section 3 where it can be.

### 5. Leaving `~/anfs-dev`

Not for an agent. In the log, for the person: once `machine-images` is merged, `git worktree
remove` the three checkouts under `~/anfs-dev` and delete the directory. Nothing reads the old
default after this plan.

### 6. Documentation

`anfs-dev`'s CLAUDE.md (or README if it has none): a "Working on the toolkit" section (the
layout, the commands, the isolation table). core's CLAUDE.md: "The Command Line" (commands from
packages; the tools as `anfs <resource>`), and `ppm hooks` becomes `anfs hooks` wherever it is
named.

## Test Spec

core `packages/anfs/tests/registry.bats`: `anfs x` runs a stub `anfs-x` with the line's rest and
its exit status; a declared command wins over `anfs-<its name>`; `anfs package --version` and
`anfs ppm --version` are ppm's; `anfs machine` with no pmm on `PATH` exits 69 with the hint;
`anfs __complete package ins` asks ppm; help lists it; completion of the
name starts no process and completion after it asks the executable; an unknown name is still 64.

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

- [ ] `bats packages/{anfs,pmm}/tests </dev/null` in core; `bats packages/dev/tests </dev/null` in `anfs-dev`
- [ ] Two jobs at once on this Mac: `anfs work run a core -- packages/pmm/tests/e2e debian` and the
      same for `b`, started together; both pass, and `/tmp/pmm-<uid>` is untouched
- [ ] `anfs work run <job> core -- packages/anfs/tests/roundtrip debian` tests the job's trees (a
      marker file in the job's core shows up in the box)
- [ ] `anfs work remove` of a merged job leaves `git worktree list` clean in every repo

## Open Questions

1. **Decided: `anfs work`**, through the seam of section 2, which is anfs's only.
2. **Does a job get its own install to try by hand** (a shell with a temp HOME whose tools are
   the job's)? The e2e scripts already build one, and `machine-images/README.md` does it by
   hand. Plan 03 has it as the `home` box.
