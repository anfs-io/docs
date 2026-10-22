---
---

# Plan 03 — `ANFS_WORK=<job>`: a job's code on live data

Written 2026-10-21 against core main `9e8cb73` plus job `wsm-03`, linker plan 02
(`link_dirs:`) and plan 01 of this unit, all merged first (unit decision 15). Independent of plans
02 and 04.

## Context — read these files first

The job is `anfs work new <job> core stack anfs-dev user`: core for `work.sh`, `paths.sh` and its
tools; stack and anfs-dev for their tools' headers and `anfs work run`; user for the one
preamble line in `pjm` (decision 12, nothing else of the user's repo). Paths are core's unless
they name another repo.

- `units/config.md` — decisions 3, 11, 12, 15; "Decided here" (the hand-over, a missing job); "Conflicts"
  (one root; `bash.sh` before the preamble); plan 01's `log.md`
- `packages/anfs/home/.local/lib/anfs/bash.sh` — the bootstrap set (3.2), how it re-execs `"$0"`
- `packages/anfs/home/.local/lib/anfs/paths.sh` after plan 01 — `ANFS_LIB_DIR`, `ANFS_BIN_DIR`
- Every entry script's header, down to its last `source`:
  core `packages/{anfs,ppm,pcm,pmm,psm,wsm}/home/.local/bin/*` (ppm's `PPM_LIB_DIR` and its loop
  over `lib/ppm/*.sh`, its `link.sh` fallback; pcm's `pcm_lib_dir`; pmm's `pmm_lib_dir`);
  anfs-dev `packages/dev/home/.local/bin/{anfs-work,anfs-hooks}`; stack
  `packages/{op/…/opm,remotefs/…/remotefs,ssh/…/ssh-key,tailscale/…/tsm,pikvm/…/pikvm,ventoy/…/ventoy,obsidian/…/ovm,ruby/…/gem-local,git/…/git-account,rpi-imager/…/rpi-imager,firstmate/…/firstmate}`
  (tsm's and rpi-imager's `load_libs`/`load_platform_lib`: installed dir first, then next to the
  script); user `packages/pjm/home/.local/bin/pjm`
- `packages/anfs/home/.local/bin/anfs` — how it execs a tool (`exec "$REPLY" …`, the
  `anfs <name>` seam to `anfs-<name>`)
- anfs-dev `packages/dev/home/.local/bin/anfs-work` — `_job_env`, `cmd_run`, `cmd_env`, the job
  layout (a worktree per repo, named by alias); `tests/work.bats`
- `packages/anfs/tests/{bash,isolation}.bats`; the test helpers that set `HOME`
  (`packages/*/tests/helper.bash`, stack's, anfs-dev's)
- linker plan 02's `log.md`: which library directories are now one link (`lib/anfs`, `lib/pcm`,
  `lib/pmm`, stack's `lib/tsm`, `lib/rpi-imager`) and that `lib/ppm` stays per file; the
  `link.sh` fallbacks it deleted

## Overview

`ANFS_WORK=<job> pcm list` runs the job's pcm, with the job's libraries, against the live
install's sources, config and data. A tool the job does not have runs as installed (with the
job's `lib/anfs` when the job has core). The switch is in the environment only, and says once on
stderr that it is on.

## Implementation

### 1. The preamble (every entry script, identical)

Directly after the shebang's comments and `set -euo pipefail`, before `bash.sh`:

```bash
# A job's copy of this tool runs instead when ANFS_WORK names one (lib/anfs/work.sh)
[ -z "${ANFS_WORK:-}" ] || [ ! -f "$HOME/.local/lib/anfs/work.sh" ] || . "$HOME/.local/lib/anfs/work.sh"
```

POSIX test syntax (it runs under macOS's bash 3.2). Nothing else is evaluated before it. A
`registry.bats` test holds every entry script listed above to this exact line (core's directly;
stack's and anfs-dev's through a list in the test, read from their repos when present).

### 2. `lib/anfs/work.sh` (bootstrap set: bash 3.2, like `bash.sh`)

Sourced only when `ANFS_WORK` is set. Functions and a call; nothing printed on the normal path but
the one line.

- **Only the installed copy hands over**: if `$0`'s directory is not `$HOME/.local/bin` (as
  given, or with `$HOME` canonical), return. So a job's copy, a test's copy and a checkout run by
  path never do; no marker variable is needed against loops.
- **The job**: `${ANFS_WORK_HOME:-$HOME/work}/$ANFS_WORK` (`ANFS_WORK_HOME` from the environment
  only). Missing (`.job.yml` absent): `anfs: no job $ANFS_WORK in <dir> (anfs work list)`, exit 64.
  `ANFS_WORK` with a `/` or starting with `.`: same message, 64.
- **The job's copy** (`anfs_work_map <path>`, also used by `paths.sh`): read the link at `$0`
  (one `readlink`, no `-f`); its text holds `…/sources/<alias>/<rest>` (relative or absolute).
  The copy is `<job>/<alias>/<rest>`. No link, no `sources/` in it, no worktree `<alias>` in the
  job, or no file there: no copy.
- **The line**: `anfs: running job <job>` on stderr, once per run: printed when
  `ANFS_WORK_SAID` is not `<job>`, which is then exported (children of this run stay quiet; a new
  run from the shell prints again). Printed whether or not this tool has a copy, since `lib/anfs`
  may still be the job's (section 3).
- **The exec**: `exec "<copy>" "$@"`. The copy runs its own preamble, which returns (its `$0` is
  not in `~/.local/bin`), then `bash.sh` (the installed one, which may re-exec the copy under
  bash 5: still the copy), then `paths.sh`.
- No copy: return, and the installed tool goes on.

### 3. One root for libraries (`paths.sh`)

- `ANFS_LIB_ROOT="$HOME/.local/lib"` is the one place the installed library path is written.
  `ANFS_LIB_DIR` (exists) is `$ANFS_LIB_ROOT/anfs`.
- **Hand-over first**: at the top of `paths.sh`, when `ANFS_WORK` is set, the job exists, and
  `anfs_work_map` of this file's own installed path (`$ANFS_LIB_ROOT/anfs/paths.sh`) gives a copy
  that is not this file, `paths.sh` sources that copy and returns. The job's `paths.sh` sets
  `ANFS_LIB_DIR` to its own directory. (`work.sh` provides `anfs_work_map`; `paths.sh` sources it
  when needed. Without a job, `paths.sh` does nothing new.)
- **`anfs_lib_dir <name>`**: `$ANFS_LIB_ROOT/<name>`, or under a job its job copy. Every library
  directory but `lib/ppm` is one link (linker 02's `link_dirs`), so the mapping is one
  `anfs_work_map` of the directory link: `sources/<alias>/<path>` becomes the job's
  `<alias>/<path>` when the job has that repo, else the installed directory. Used by pcm, pmm, tsm
  for their own libraries.
- **`lib/ppm`, the one per-file directory** (core/ppm, anfs-dev/dev and stack/ruby all write
  there): `anfs_lib_files ppm` lists the `*.sh` to source: each installed file, or its job copy
  when the job has its repo and the file; then each `*.sh` in the job's `core/packages/ppm/home/.local/lib/ppm/`
  not already listed (a file new in the job). Sorted by basename, as the glob is today. It is the
  only caller; the function takes the name so a second shared directory would need no new code.
- Entry scripts: `ANFS_LIB="$HOME/.local/lib/anfs"` goes. They source
  `"$HOME/.local/lib/anfs/bash.sh"` and `"$HOME/.local/lib/anfs/paths.sh"` by literal path (the
  two entry points: the bootstrap and the hand-over), then everything from `"$ANFS_LIB_DIR"`.
  Their "missing or out of date" checks test `$HOME/.local/lib/anfs/paths.sh` and, after it, the
  files they need in `$ANFS_LIB_DIR`. (pcm's check calls `anfs_warn` before any library defines it:
  an `echo … >&2` as the others do.)
- ppm: `PPM_LIB_DIR` is `$ANFS_LIB_ROOT/ppm` (still the installed directory, where `file.sh` and
  the extensions look for siblings) and its loop is over `anfs_lib_files ppm`; every
  `$HOME/.local/lib/anfs/<file>` in it becomes `$ANFS_LIB_DIR/<file>`.
- pcm, pmm: `pcm_lib_dir`/`pmm_lib_dir` become `anfs_lib_dir pcm|pmm` (an explicit
  `PCM_LIB_DIR`/`PMM_LIB_DIR` still wins, for tests); their loops glob that directory.
- stack `tsm`: its candidate list starts with `$(anfs_lib_dir tsm)` instead of
  `$HOME/.local/lib/tsm`. `rpi-imager` does not source `lib/anfs`: its candidate list puts "next
  to the script" before `$HOME/.local/lib/rpi-imager`, which is right both installed (the link
  resolves into the source, the same files) and as a job's copy.
- `firstmate` and `bip39` (ruby) take no preamble beyond the line (firstmate is bash: it gets the
  line; bip39 is Ruby: none, said in the log).

### 4. Where `ANFS_WORK` must not reach

- `anfs work run` and `anfs work env`: `run` unsets `ANFS_WORK` and `ANFS_WORK_SAID` in the
  command's environment; `env` prints `unset ANFS_WORK` first. Tests always run the job's code by
  path, never through the switch.
- Every test helper that sets a temp `HOME` (core's `packages/*/tests/helper.bash` and the bats
  files that set `HOME` themselves, stack's, anfs-dev's) unsets `ANFS_WORK`. `isolation.bats`
  gains a check: a suite that sets `HOME` also unsets `ANFS_WORK` (or its helper does).
- `anfs_load_conf` already refuses it from a file (plan 01); `anfs config list` shows it (plan
  02's row; if 02 has not merged, nothing here adds it).

### 5. `anfs work` itself

- `anfs work show <job>` ends with a line: `run its tools on live data: ANFS_WORK=<job> <tool> …`.
- `anfs work list` gains nothing (the switch is per shell, not per job).
- `cli_help_epilogue`: one sentence on `ANFS_WORK`.

### 6. Documentation

Core `AGENTS.md`: a section "Running a job's code: ANFS_WORK" (the preamble line, `work.sh`'s
rules, the library mapping, what it is for and that data is live; that tests never use it);
"Lib Structure" (`work.sh`, bootstrap set; `paths.sh`'s `ANFS_LIB_ROOT`, `anfs_lib_dir`,
`anfs_lib_files`); "Development" (a new entry script starts with the preamble line).
`ROADMAP.md`'s "For an agent running this list" is the person's to change, not this plan's: the
log suggests one bullet. anfs-dev `packages/dev/README.md`: `ANFS_WORK`.

## Test Spec

- `packages/anfs/tests/work.bats` (new; temp `HOME`, a fake sources tree and a fake
  `~/work/<job>/core/`, tools linked into the temp `~/.local/bin` as ppm links them, with
  `HOME`'s `.local/lib/anfs` the job's core under test):
  - With `ANFS_WORK` unset, the installed copy runs (each fake tool prints its own path).
  - `ANFS_WORK=j`: the job's copy runs, with `"$@"` intact (spaces, empty args); stderr has
    exactly one `anfs: running job j`.
  - A tool whose package the job lacks runs as installed, and still prints the line once.
  - The job's copy, run by path with `ANFS_WORK=j`, does not exec anything (no loop).
  - `ANFS_WORK=nosuch`: exit 64, the message; `ANFS_WORK=../x`: 64.
  - Nested: a job tool that runs another installed tool: the second hands over too, no second line.
  - `paths.sh` under `ANFS_WORK=j` sets `ANFS_LIB_DIR` to the job's `lib/anfs`; `anfs_lib_dir pcm`
    maps the directory link to the job's, and to the installed one when the job lacks core;
    `anfs_lib_files ppm` returns the job's copy of `core.sh`, the installed `container.sh`
    (anfs-dev's, not in the job), and a file only in the job.
  - Under `/bin/bash` (3.2, macOS; skipped elsewhere, as `bash.bats` does): the preamble and
    `work.sh` run and hand over before `bash.sh` re-execs.
- `bash.bats`: `work.sh` parses under the old bash (the bootstrap set).
- `registry.bats`: every core entry script has the preamble line first after `set -euo pipefail`;
  none assigns `ANFS_LIB=` or names `$HOME/.local/lib/` except the two literal `source` lines.
- anfs-dev `work.bats`: `anfs work run` with `ANFS_WORK` exported runs the command without it;
  `env` starts with `unset ANFS_WORK`.
- `isolation.bats`: the new check.
- stack: each tool's suite passes with the new header; a grep in stack's tests (or the log) shows
  every entry script on `lib/anfs` has the line.

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing that
boots a VM or a box runs, and nothing is installed on this host. pmm's and pcm's suites stay off;
the log says so.

- [ ] While working, the file you touch: `bats packages/anfs/tests/work.bats`,
      `bats packages/anfs/tests/{bash,registry,isolation}.bats`
- [ ] `bats packages/anfs/tests`, `bats packages/ppm/tests`, `bats packages/wsm/tests`,
      `bats packages/psm/tests`
- [ ] anfs-dev: `anfs work run <job> anfs-dev -- bats packages/dev/tests </dev/null`
- [ ] stack: `anfs work run <job> stack -- env ANFS_TEST_LIB=<job's core>/packages/anfs/home/.local/lib/anfs
      bats packages/<tool>/tests` for each tool whose header changed and has tests
- [ ] `grep -rn 'ANFS_LIB=\|\$HOME/\.local/lib/' packages/*/home/.local/bin` in core, stack and
      anfs-dev: only the two literal `source` lines per script, and `rpi-imager`'s candidate
      (user's `pjm` keeps its `ANFS_LIB=` line: it gets the preamble only, so under a job it
      hands over to a job's `pjm` but its libraries stay the installed ones)
- [ ] user: `anfs work run <job> user -- bats packages/pjm/tests` (the one line changed)
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`): `ppm install -f core/anfs core/ppm core/pcm
  core/pmm core/psm core/wsm anfs-dev/dev user/pjm` and each installed stack tool; then with a job that
  has a visible change in pcm: `ANFS_WORK=<job> pcm help` (the change, and `anfs: running job
  <job>` once); `ANFS_WORK=<job> ventoy help` (stack not in the job: runs as installed, with the
  job's `lib/anfs`); `ANFS_WORK=nosuch pcm list` exits 64; `anfs work run <job> core -- env |
  grep ANFS_WORK=` prints nothing while `ANFS_WORK` is exported in the shell.
- Deferred to the end of the roadmap (named in `log.md`): `roundtrip debian` (Linux readlink
  forms, the preamble under Debian's bash), `roundtrip-macos`.
