---
status: built, awaiting review
started_at: "2026-10-19"
completed_at: "2026-10-19T12:00:00-04:00"
deviations: cli_section in cli.sh. A sixth job variable, ANFS_WORK_NUMBER, and ANFS_WORK_TMP for where a job's per-number directories go. anfs work run takes a repo (run <job> core -- ...). pmm's suites use a directory inside a job's PMM_SOCK_DIR; image suites take ports 50 above the base. ROUNDTRIP_SRC_ROOT defaults to the directory the running core is in. A missing tool is looked for in ~/.local/bin. anfs hooks alone prints help. sh/anfs.sh sends all six tools through their own functions. Only roundtrip-macos takes the tart lock. remove refuses files in no worktree, and main/master are not job names. Under bats pmm refuses the live socket directory. Full list below.
summary: Plan 02 is built on branch anfs-work in core (5 commits), anfs-dev (3) and docs; not pushed, not merged, not installed. The review's blocker (remove -f deleting an unmerged branch unlisted) and both should-fixes are fixed, with most minors. Full core run 478 passed, 0 failed (40 files); anfs-dev 26 of 26. Two jobs ran pmm's bats suites at once from an isolated HOME (113 and 113, separate socket dirs, port bases and TMPDIRs, /tmp/pmm-501 untouched). Not verified - every VM and box check (deferred), anfs work against the real sources, zsh.
---

# Log: dev-source/02-work

Built 2026-10-19 on the branch `anfs-work` in core, anfs-dev and docs (`~/work/anfs-work/`, the
last job set up by hand). Not pushed, not merged, nothing installed. Starting points: core
15d1aaa (60c9c1b on the branch is `tests/all`, not this plan), anfs-dev e570f58, docs 781d9cc.

## Read this first

1. **`anfs work run` takes a repo**: `anfs work run <job> core -- bats packages/pmm/tests`. Without
   one it runs in the job's directory, which holds the worktrees and no `packages/`, so a test
   command written as the plan had it (`anfs work run a -- packages/...`) finds nothing. ROADMAP's
   agent section, the plan's Verification lines and CLAUDE.md now say `core`.
2. **`anfs work remove` refuses more than the plan said**: files in the job's directory that no
   worktree holds (notes, logs) are refused like a dirty worktree, and listed under `-f`.
3. **Under bats, pmm refuses `/tmp/pmm-<uid>`** (exit 69, before any command), whether
   `PMM_SOCK_DIR` names it or is unset. A suite of your own that never set `PMM_SOCK_DIR` now
   fails instead of reaching the live machines' sockets.
4. After the merge, on this Mac: see "For the person".

## Commits

core:

- `f958857` — the tools as `anfs <resource>`, commands from packages, `cli_section`, `sh/anfs.sh`,
  the `anfs-*-manager` trial links deleted, `bash.bats` on core's own entry points
- `b53b705` — the isolation of section 3: pmm's helper and e2e, the round trips, the harness, the
  tart lock, `isolation.bats`
- `d1fb45c` — CLAUDE.md
- `632bdad` — review fixes: the shell function routes all six tools, no fork per package command
  at start, `isolation.bats` catches a `$uid` of the test's own and tests its own pattern
- `06ad351` — review fix: pmm refuses the live socket directory under bats; CLAUDE.md

anfs-dev:

- `bfab4ac` — `anfs work` and `anfs hooks` as `anfs-<name>` executables, `hooks.sh` out of
  `lib/ppm`, the package's tests (work, hooks, the bash floor of its entry points)
- `900c2e4` — `anfs work` through the real anfs: the seam, help, the shell function's cd
- `0bb0e70` — review fixes: remove counts the job's branch, refuses files in no worktree, deletes
  `PMM_SOCK_DIR` too; `run <job> [repo]`; the number under a lock; `main`/`master` refused

docs: `1a75a0b` (the agent section, the first log), then one commit with this log, the roadmap
line, the plan's Verification commands and the agent section's `run ... core --`.

## What was built

- **`anfs <name>` runs `~/.local/bin/anfs-<name>`** for a name anfs does not declare (an alias
  counts as declared). The names are read from the file names when anfs starts (a glob, no
  process, no subshell) and declared under a help section "From packages", so help and name
  completion need nothing else. `help <name>`, `<name> --help` and `__complete <name> ...` are the
  executable's. `-d`, `--trace` and `--json` are passed on as leading arguments.
- **The tools**: `package|container|machine|skill|space|vault`, aliases `ppm|pcm|pmm|psm|wsm|pvm`,
  under "Tools" in help. All six names were free (`anfs help` had src, iso, install, onboard,
  implode, callbacks, log, completion, help).
- **`anfs work`** as the plan's table says, plus `run`'s repo argument, and **`anfs hooks
  status|install|uninstall`**.
- **Isolation**: the variables of section 3, honoured by pmm's helper and e2e, the round trips and
  the harness; `isolation.bats` greps for what no test may do, and pmm itself refuses the live
  socket directory under bats.

## Deviations from the plan

- **`cli_section` is new in `cli.sh`.** Help's "Tools" and "From packages" needed a way to list
  top-level commands under a heading; it is a declaration like `cli_group`, usable by any tool.
- **A sixth variable, `ANFS_WORK_NUMBER`**: the harness needs the number for its box names and
  the round trip for the box it execs into. `anfs work list` has a `NUMBER` column (`number` was
  already one of the plan's JSON keys).
- **`ANFS_WORK_TMP`** (default `/tmp`) is where a job's `PMM_SOCK_DIR` and `TMPDIR` go, so the
  tests can point it at a temp dir. `remove` deletes both.
- **`anfs work run <job> [repo] -- <cmd>`**: the plan had only the job's directory (review
  should-fix 1, below).
- **`remove` refuses files in no worktree**, and **`main`, `master` and `HEAD` are not job names**
  (64); a repo whose main is the job's name is refused at `new`/`add` (review, below).
- **The number is chosen under a lock**, `$ANFS_WORK_HOME/.numbers.lock` (a `mkdir`, holding the
  pid; a dead holder is taken over, a live one for 10 s is 75).
- **pmm's suites under a job use a directory inside the job's `PMM_SOCK_DIR`** (`<dir>/bats`), and
  the e2e a fresh `<dir>/e2e.*`: both implode what they use. Without a job they are as before
  (`/tmp/pmm-bats-<uid>`, `/tmp/pmm-e2e.*`).
- **Under bats, pmm refuses the live socket directory** (`sock_dir_path` in `lib/pmm/core.sh`,
  checked by the `pmm` entry point before any command and by `implode`). Not in the plan: the
  review found the grep could not see a suite that never sets `PMM_SOCK_DIR`.
- **The port base**: machine suites start at the job's `PMM_SSH_PORT_BASE` (default 42300), image
  suites 50 above it (default was a fixed 42200). Not below: under job 1 that is 2222, the live
  default, and a provision test failed there on this Mac because the running qemu machine
  listens on 2222. The suites' port assertions are now relative to the base.
- **`ROUNDTRIP_SRC_ROOT`'s default is no longer `~/anfs-dev`**: it is the directory the running
  core checkout is in (a job's directory when run from `~/work/<job>/core`). Of core and stack,
  one the root does not hold comes from the installed sources (`roundtrip-sources`).
- **A tool that is not installed** is looked for in `~/.local/bin` (as `anfs install` runs tools),
  not on PATH: the test points HOME at a temp dir instead of stripping PATH.
- **`anfs hooks` alone prints its help**; `ppm hooks` alone was the status. `status` is a command.
- **`sh/anfs.sh`** sends each of the six tools through the tool's own shell function when there is
  one (`ppm`'s cd and reload, `wsm`'s and `pmm`'s cd), and `anfs <name> cd` to `anfs <name> path`.
  The plan asked only for the latter.
- **The tart lock is taken by `roundtrip-macos` only**: no other test in core uses tart (pmm's e2e
  points `TART_HOME` at a temp dir). It is a directory (`mkdir` is atomic) holding the pid and
  job; a holder that is no longer running is taken over; a live one is exit 75.
- **The harness's `bootstrap` snapshot images are not per job** (`localhost/anfs-test:<distro>-…`):
  two jobs building them for the first time at once would race. Only the boxes are named per job.
- core's CLAUDE.md "Repository Layout" no longer lists `packages/dev` (moved by hand already),
  though the rest of that file's `core/dev` references are plan 01's to rewrite.
- `bash.bats` in core checks core's tools (pvm added), any `anfs-*` in core's packages, pcm's
  edges and pmm's provisioners, and requires at least seven; anfs-dev's `bash.bats` checks the
  hook, `anfs-work` and `anfs-hooks`. This fixed the one test that failed on main (it named the
  pre-commit hook, which had moved to anfs-dev).

## Review and fixes

Each finding was reproduced or read in the code before it was changed.

| Severity | Finding | What became of it |
| --- | --- | --- |
| blocker | `remove` read unmerged work from the worktree's HEAD; a worktree that was gone or on another branch hid the branch's commits, and `-f` deleted them unlisted | **Fixed** (0bb0e70). `_repo_state` counts commits not in main on `refs/heads/<job>` in the origin plus the worktree's HEAD; `remove` lists them with `git log` over the same revisions. Tests: worktree deleted by hand (refused, `-f` without `-y` lists `change wip-a` and asks, `-f -y` removes); worktree switched to another branch (refused). A missing worktree is not itself a problem: what it held is already gone, and its branch is counted |
| should-fix | ROADMAP's `anfs work run <job> -- bats packages/...` ran in the job's directory, which has no `packages/` | **Fixed**: `run <job> [repo] -- <cmd>` runs in that repo's worktree (an unknown repo is 1). ROADMAP, the plan's Verification, CLAUDE.md and the README say `core`. Test added |
| should-fix | `remove` deleted the whole job directory, files of the person's included, even without `-f` | **Fixed**: what is in the directory besides `.job.yml` and the repos' worktrees is refused without `-f` and listed with it. Test with a `NOTES.md` |
| minor | job numbers could collide between two `new` at once; `remove` left `PMM_SOCK_DIR` behind | **Fixed**: the number is taken under a lock; `remove` deletes `PMM_SOCK_DIR` too. Test: four `new` in the background get 1 2 3 4; it fails all of three runs with the lock taken out |
| minor | forks in completion: `$(anfs_display ...)` per package command at every anfs start, `basename "$(dirname ...)"` in `complete_job`/`complete_repo` | **Fixed** with parameter expansion |
| minor | `sh/anfs.sh` routed only `anfs package`; `anfs space cd`, `anfs machine cd` reached the binaries | **Fixed**: all six go through the tool's function when it has one. Test in `registry.bats` (stub functions; PATH limited so the host's pcm is not found). **Left**: a global option first (`anfs -d space cd x`) still skips the function (portable splitting of the saved options in sh and zsh was not worth it); documented in CLAUDE.md |
| minor | gaps in `isolation.bats` | **Fixed in part**: the socket pattern also catches `$uid`/`${uid}` (and any `*UID`/`*uid` variable), with a test of the pattern itself; the hole no grep can see is closed by pmm's own refusal under bats (test in `machine.bats`). **Not changed**: a single-quoted `'~/...'` is not the real home (the shell does not expand it; the tool expands it against the test's HOME, as `wsm/tests/install.bats` does on purpose), so it stays allowed. anfs-dev's suites stay uncovered: they are in another repo; the comment says so |
| minor | a job named after an existing branch, such as `main`, took it over and `remove` could delete it; a failing `branch -d` aborted half way | **Fixed in part**: `main`, `master` and `HEAD` are refused (64), and so is a job named as a repo's main at `new`/`add`. `remove` deletes with `-D` once merged into main has been checked (so the origin's HEAD does not matter) and warns instead of aborting. **Left**: whether the branch was created by the job is not recorded; adopting an existing branch is how a job is resumed |
| minor | docs still name `core/dev` (CLAUDE.md, the four lib headers in anfs-dev) | **Left** for plan 01, which rewrites them with the rest of the move |

Not run while fixing: one probe of the reviewer's (`probe.sh`, in its own scratchpad); the
scenarios were rebuilt as bats tests instead.

## Test results

macOS 26 arm64, bats-core, on core 06ad351 and anfs-dev 0bb0e70.

- While fixing: `bats packages/dev/tests` (26 of 26), `bats packages/anfs/tests` (154 of 154),
  `bats packages/pmm/tests` (113 of 113).
- **The full run, once**: `packages/anfs/tests/all` in core: **478 passed, 0 failed, 40 files, 8
  at a time, 139 s**. `bats packages/dev/tests </dev/null` in anfs-dev: **26 of 26**. Nothing
  failed, so the full run was not repeated.
- **Two jobs at once**, in an isolated HOME (`/private/tmp/claude-501/v3`) whose sources `core`
  and `anfs-dev` were local clones of the worktrees (`anfs-work` renamed `main`), with
  `ANFS_WORK_HOME` and `ANFS_WORK_TMP` inside it:
  - `anfs work new a core` and `anfs work new b core` started together: numbers 1 and 2.
  - `anfs work run a core -- bats packages/pmm/tests </dev/null` and the same for `b`, started
    together in the background: **113 of 113 each, both exit 0**.
  - Their variables differ: `PMM_SOCK_DIR` `…/wt/pmm-work-501-1` and `-2`, `PMM_SSH_PORT_BASE`
    2322 and 2422, `TMPDIR` `…/wt/anfs-work-501-1` and `-2`; each socket directory held its own
    `bats/` afterwards.
  - `/tmp/pmm-501`: the same two entries (`dj-tester.ga`, `dj-tester.qmp`, Oct 16 07:05) and the
    same directory mtime before and after.
- **remove of a merged job**: job `m` with `core` and `anfs-dev`, a commit in each; `remove`
  refused (`core: 1 commits not in main; anfs-dev: 1 commits not in main`, exit 1); after a
  fast-forward merge into each clone's main, `list` said `merged` and `remove` exit 0. `git
  worktree list` in each clone was the clone alone, the branch `m` was gone, and `ANFS_WORK_HOME`
  and `ANFS_WORK_TMP` were empty after `a` and `b` were removed too.
- **By hand, same HOME**: `anfs package --version` and `anfs ppm --version` both `ppm 0.2.4`, exit
  0; `anfs machine image list` listed core's four images (unbuilt), exit 0; `anfs help` has the
  Tools section with the six and their aliases, and "From packages: work"; with `pmm` taken out
  of `~/.local/bin` (and none on PATH), `anfs machine list` said `anfs: pmm is not installed
  (ppm install core/pmm)`, exit 69.

## Verification (the plan's list)

- [x] `bats packages/{anfs,pmm}/tests </dev/null` in core; `bats packages/dev/tests </dev/null` in
      anfs-dev (and the full `packages/anfs/tests/all`)
- [ ] **Deferred** (no VMs, decided 2026-10-19): two jobs' `pmm` e2e at once on this Mac, with
      `/tmp/pmm-<uid>` untouched. Done instead with the pmm bats suites, above
- [ ] **Deferred** (no boxes): `anfs work run <job> core -- packages/anfs/tests/roundtrip debian`
      showing a marker file from the job's core in the box
- [x] `anfs work remove` of a merged job leaves `git worktree list` clean in every repo: in an
      isolated HOME over clones of the worktrees (not the real sources)

## For the person

- After the merge, here: `ppm install anfs-dev/dev` again, so `~/.local/bin/anfs-work` and
  `anfs-hooks` are linked and `hooks.sh` leaves `~/.local/lib/ppm`. Then `anfs hooks install` (it
  replaces `ppm hooks install`; the `core.hooksPath` values already set stay valid).
- core's changed files (`anfs`, `sh/anfs.sh`, `pmm`, `lib/pmm/core.sh`, `implode.sh`) are linked
  already, so they are live as soon as the merge reaches the clone under `sources/`; a shell
  started after that gets the new `anfs` function. No core package adds a file.
- Section 5: once nothing is left to merge from it, `git worktree remove` the three checkouts
  under `~/anfs-dev` and delete the directory. Nothing reads it any more.
- The testing review (ROADMAP "Last") decides how the deferred checks above are run.

## Not verified

- Two jobs' pmm e2e at once (real VMs), and the round trip marker check (a box): deferred.
- `anfs work` against the real sources under `~/.local/share/anfs/sources/` (only clones of the
  worktrees in an isolated HOME, and bare repos in `work.bats`).
- A space's repository (`docs`) through the real installed wsm spaces: only `work.bats`' fixture.
- `anfs work cd` and the `anfs <resource>` routing under zsh: the tests run `sh/anfs.sh` in bash.
- `anfs work new` wiring the hooks into a real source (the isolated HOME had no hooks directory;
  `work.bats` covers it).
- The tart lock under two real `roundtrip-macos` runs.
- `anfs hooks install|uninstall` against the real repos.
