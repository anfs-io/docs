# Log: dev-source/02-work

Built 2026-10-19 on the branch `anfs-work` in core, anfs-dev and docs (`~/work/anfs-work/`, the
last job set up by hand). Not pushed, not merged. Starting points: core 15d1aaa (60c9c1b on the
branch is `tests/all`, not this plan), anfs-dev e570f58, docs 781d9cc.

## Commits

- anfs-dev `bfab4ac` — `anfs work` and `anfs hooks` as `anfs-<name>` executables, `hooks.sh` out
  of `lib/ppm`, the package's tests (work, hooks, the bash floor of its entry points)
- anfs-dev `900c2e4` — `anfs work` through the real anfs: the seam, help, the shell function's cd
- core `f958857` — the tools as `anfs <resource>`, commands from packages, `cli_section`,
  `sh/anfs.sh`, the `anfs-*-manager` trial links deleted, `bash.bats` on core's own entry points
- core `b53b705` — the isolation of section 3: pmm's helper and e2e, the round trips, the harness,
  the tart lock, `isolation.bats`
- core `d1fb45c` — CLAUDE.md
- docs — ROADMAP.md's agent section, this log

## What was built

- **`anfs <name>` runs `~/.local/bin/anfs-<name>`** for a name anfs does not declare (an alias
  counts as declared). The names are read from the file names when anfs starts (a glob, no
  process) and declared under a help section "From packages", so help and name completion need
  nothing else. `help <name>`, `<name> --help` and `__complete <name> ...` are the executable's.
  `-d`, `--trace` and `--json` are passed on as leading arguments.
- **The tools**: `package|container|machine|skill|space|vault`, aliases `ppm|pcm|pmm|psm|wsm|pvm`,
  under "Tools" in help. All six names were free (`anfs help` had src, iso, install, onboard,
  implode, callbacks, log, completion, help).
- **`anfs work`** as the plan's table says, and **`anfs hooks status|install|uninstall`**.
- **Isolation**: the variables of section 3, honoured by pmm's helper and e2e, the round trips and
  the harness; `isolation.bats` greps for what no test may do.

## Deviations from the plan

- **`cli_section` is new in `cli.sh`.** Help's "Tools" and "From packages" needed a way to list
  top-level commands under a heading; it is a declaration like `cli_group`, usable by any tool.
- **A sixth variable, `ANFS_WORK_NUMBER`**: the harness needs the number for its box names and
  the round trip for the box it execs into. `anfs work list` has a `NUMBER` column (`number` was
  already one of the plan's JSON keys).
- **`ANFS_WORK_TMP`** (default `/tmp`) is where a job's `PMM_SOCK_DIR` and `TMPDIR` go, so the
  tests can point it at a temp dir. `remove` deletes the job's `TMPDIR` too.
- **pmm's suites under a job use a directory inside the job's `PMM_SOCK_DIR`** (`<dir>/bats`), and
  the e2e a fresh `<dir>/e2e.*`: both implode what they use, and the job's own directory should
  outlive one run. Without a job they are as before (`/tmp/pmm-bats-<uid>`, `/tmp/pmm-e2e.*`).
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
- **`sh/anfs.sh`**: besides `anfs <name> cd`, `anfs package ...` goes through ppm's shell
  function, so `anfs package cd` and the reload after an install behave as `ppm`'s do.
- **The tart lock is taken by `roundtrip-macos` only**: no other test in core uses tart (pmm's e2e
  points `TART_HOME` at a temp dir). It is a directory (`mkdir` is atomic) holding the pid and
  job; a holder that is no longer running is taken over; a live one is exit 75.
- **The harness's `bootstrap` snapshot images are not per job** (`localhost/anfs-test:<distro>-…`):
  two jobs building them for the first time at once would race. Only the boxes are named per job.
- core's CLAUDE.md "Repository Layout" no longer lists `packages/dev` (moved by hand already),
  though the rest of that file's `core/dev` references are plan 01's to rewrite.
- `bash.bats` in core checks core's tools (pvm added), any `anfs-*` in core's packages, pcm's
  edges and pmm's provisioners, and requires at least seven; anfs-dev's `bash.bats` checks the
  hook, `anfs-work` and `anfs-hooks`.

## Verification

- [x] `bats packages/anfs/tests </dev/null` (152) and `bats packages/pmm/tests </dev/null` (112)
      in core; pmm's port-sensitive files again under a job's `PMM_SSH_PORT_BASE=2322` and a temp
      `PMM_SOCK_DIR`; `bats packages/dev/tests </dev/null` (21) in anfs-dev, against the job's core
- [x] By hand, in a temp HOME whose sources were clones of the worktrees: `anfs work new try core
      anfs-dev`, `list` (and `--json`), `show`, `cd` through `sh/anfs.sh`, `run`, `remove`; the
      clone's `git worktree list` was clean afterwards
- [ ] **Deferred** (no VMs, decided 2026-10-19): two jobs' `pmm` e2e at once on this Mac, with
      `/tmp/pmm-<uid>` untouched
- [ ] **Deferred**: `anfs work run <job> -- packages/anfs/tests/roundtrip debian` showing a marker
      file from the job's core in the box
- [ ] `anfs work remove` of a merged job in the real sources: after the merge, on the first real
      job (work.bats covers it with bare repos)
- [ ] The full `packages/anfs/tests/all`: the last step's, not run here

## For the person

- After the merge: `ppm install anfs-dev/dev` again here, so `~/.local/bin/anfs-work` and
  `anfs-hooks` are linked (and `hooks.sh` leaves `~/.local/lib/ppm`).
- Section 5: once nothing is left to merge from it, `git worktree remove` the three checkouts
  under `~/anfs-dev` and delete the directory. Nothing reads it any more.
- The testing review (ROADMAP "Last") decides how the deferred checks above are run.
