---
plan: config/03-work-switch
job: config-03
date: 2026-10-22
---

# Log: config/03-work-switch

## The plan against current main

Checked before starting (core `533935e`, stack `2f989ee`, anfs-dev `683331f`, user `710b294`):
linker/02 has merged as the plan assumed. On this Mac `~/.local/lib/{anfs,pcm,pmm}` are one link
each into `sources/core/...`, `lib/ppm` is per file (core/ppm, anfs-dev/dev's `container.sh`,
`move.sh`, `user.sh`, `vm.sh`, stack/ruby's `ruby.sh`), `platform.sh` is in `lib/anfs`. Every
installed tool link and library link text holds `.../sources/<alias>/...` (relative). Config 01's
loader already refuses `ANFS_WORK`, `ANFS_WORK_HOME`, `ANFS_WORK_TMP` in a file.

## What was built

- **`lib/anfs/work.sh`** (new, bash 3.2, bootstrap set): `anfs_work_job` (the job directory,
  `${ANFS_WORK_HOME:-~/work}/$ANFS_WORK` with a leading `~/` expanded; a `/` or a leading dot, or no
  `.job.yml`: not a job), `anfs_work_where` (one `readlink`, no `-f`: `.../sources/<alias>/<rest>`
  to `<job>/<alias>/<rest>` when the job has the worktree), `anfs_work_map` (that, and the file
  exists), `anfs_work_handover` (only `$0` in `$HOME/.local/bin`, `$HOME` as given or canonical;
  missing job: `anfs: no job <job> in ~/work (anfs work list)`, exit 64; `anfs: running job <job>`
  once, `ANFS_WORK_SAID` exported; `exec` the copy with `"$@"`). Ends with `true`, so the preamble's
  `||` list never trips `set -e`. `_ANFS_WORK_LOAD_ONLY=1` sources it for the functions alone.
- **The preamble** (two lines: a comment and the plan's line) right after the first `set -euo
  pipefail` of every entry script: core's anfs, ppm, pcm, pmm, psm, wsm; stack's opm, remotefs,
  ssh-key, tsm, pikvm, ventoy, ovm, gem-local, git-account, rpi-imager, firstmate; anfs-dev's
  anfs-work, anfs-hooks; user's pjm. bip39 (Ruby) has none.
- **`paths.sh`**: `ANFS_LIB_ROOT="$HOME/.local/lib"`; at the top, under `ANFS_WORK`, it sources
  `work.sh` (load only) from its own directory and, when it is the installed one (its
  `BASH_SOURCE` is `$ANFS_LIB_ROOT/anfs/paths.sh`) and the job has a copy of the `lib/anfs` link,
  sources the job's `paths.sh` and returns. `ANFS_LIB_DIR` is now `${BASH_SOURCE[0]%/*}`: the
  directory of the `paths.sh` that ran. New `anfs_lib_dir <name>` (prints the job's copy of the
  directory link, else `$ANFS_LIB_ROOT/<name>`) and `anfs_lib_files <name>` (per-file directories:
  each installed `*.sh` or its job copy, then the `*.sh` only the job has in the job directories
  that hold the installed ones; sorted by basename; without a job, the glob as before, no fork).
- **Entry scripts' headers**: `ANFS_LIB=` gone; the "missing" check tests
  `$HOME/.local/lib/anfs/paths.sh` (message names `~/.local/lib/anfs`; pcm's was an `anfs_warn`
  before any library, now an `echo`); `bash.sh` and `paths.sh` sourced by literal path; a script
  that checked more files (`table.sh`, `cli.sh`, `iso.sh`) checks them in `$ANFS_LIB_DIR` right
  after `paths.sh`; everything else from `$ANFS_LIB_DIR`. anfs and ppm keep their optional
  `bash.sh`.
- **ppm**: `PPM_LIB_DIR="$ANFS_LIB_ROOT/ppm"` (after `paths.sh`); the loop is over `anfs_lib_files
  ppm`; every `$HOME/.local/lib/anfs/<file>` is `$ANFS_LIB_DIR/<file>`; `lib/ppm/sources.sh`
  sources lib/anfs's `sources.sh`/`resolve.sh` from `$ANFS_LIB_DIR`.
- **pcm, pmm**: `pcm_lib_dir`/`pmm_lib_dir` gone; `PCM_LIB_DIR="${PCM_LIB_DIR:-$(anfs_lib_dir pcm)}"`
  (pmm the same). The fallback to the libraries beside the script's real path went with them.
- **anfs implode** deletes `$ANFS_LIB_ROOT/anfs`, not `$ANFS_LIB_DIR` (which may be a job's
  checkout).
- Also on `$ANFS_LIB_DIR`: pmm's `anfs` provisioner (`paths.sh` literal, then `log.sh`,
  `sources.sh`); psm's and wsm's `install.sh` (`${ANFS_LIB_DIR:-$HOME/.local/lib/anfs}/skill.sh`).
- **stack**: tsm's candidate list starts (after `TSM_LIB`) with `$(anfs_lib_dir tsm)`;
  rpi-imager's puts the two "next to the script" candidates before `$HOME/.local/lib/rpi-imager`.
- **`anfs work`** (anfs-dev): `run` unsets `ANFS_WORK` and `ANFS_WORK_SAID`; `env` prints `unset
  ANFS_WORK ANFS_WORK_SAID` first; `show` ends with `run its tools on live data: ANFS_WORK=<job>
  <tool> ...`; the help epilogue has the sentence; `env`'s summary says it unsets.
- **Test helpers**: every bats file or helper that sets `HOME` unsets `ANFS_WORK ANFS_WORK_SAID`
  (core: 19 bats files at top level, pcm/pmm/wsm helpers; stack: 9 bats files, obsidian's helper;
  anfs-dev's helper). conf.bats already did.
- **Docs**: core AGENTS.md "Running a job's code: ANFS_WORK" (new, before "Development"); Lib
  Structure (`work.sh`, `paths.sh`'s new names, the rule "bash.sh and paths.sh by literal path, the
  rest from `$ANFS_LIB_DIR`", the bootstrap set, ppm's library loop); anfs-dev README "Trying a
  job's code on live data: ANFS_WORK".

## Commits (branch config-03, not pushed)

core:
- `3e975e7` ANFS_WORK: work.sh, the preamble, paths.sh, the tools, tests
- `4e0a12a` docs (AGENTS.md)
- `47fc72a` tests: iso.bats runs pmm with `PMM_LIB_DIR`; settings.bats allow-list

stack: `0d662d6` the preamble, headers, tsm, rpi-imager, tests (firstmate 0.1.3, git 0.1.4, ruby
0.1.2 bumped by the hook; the others already carry an unpushed bump)

anfs-dev: `c6fa934` anfs work run/env/show, the preamble, README, tests (dev already 0.1.15)

user: `0a74c6a` pjm: the preamble (pjm already 0.1.1 unpushed). Nothing else of user's repo.

core's anfs, pcm, pmm, ppm, psm, wsm already carry an unpushed bump (config-01's): the hook left
them. `WSM_VERSION`, `OVM_VERSION` unchanged (no command changed).

## Decisions made during the work

- **The entry-script tests are in `work.bats`, not `registry.bats`** (registry.bats is cli.sh's,
  and config-02 edits it): every core entry script has the two preamble lines right after `set
  -euo pipefail`, assigns no `ANFS_LIB=`, and names `$HOME/.local/lib/` only in the preamble, the
  `paths.sh` check and the two literal `source` lines; stack's, anfs-dev's and user's pjm have the
  preamble when their repo sits beside core (a job, or the sources directory).
- **`ANFS_LIB_DIR` is always the directory of the `paths.sh` that was sourced**, not only after a
  hand-over: the same value for every installed tool; a test that sources a checkout's `paths.sh`
  gets the checkout's.
- **The paths.sh hand-over happens only from the installed `paths.sh`** (its `BASH_SOURCE` is the
  literal installed path), the same rule as the tools'. The job's `paths.sh` re-sources its own
  `work.sh`, so the library mapping is the job's code; the tool's exec is decided by the
  installed `work.sh` (it must be: it runs before anything of the job's).
- **`anfs_lib_files` finds "new in the job" files in the job directories that already hold an
  installed file's copy** (for lib/ppm: core's `packages/ppm/.../lib/ppm` and, when the job has
  anfs-dev, dev's), not a hard-coded `core/packages/ppm` path. An installed file whose repo is in
  the job but which the job deleted stays the installed one (the plan's rule).
- A bad `ANFS_WORK` reaching `paths.sh` without the preamble (a job's copy run by path, a hook) is
  ignored there: the preamble owns the error.
- `ANFS_WORK_SAID` is not in `anfs_conf_env_only` (a settings line naming it only silences the
  line); listed with the job variables in settings.bats's "not settings".

## Tests (all through `anfs work run config-03 …`, `</dev/null`)

- While working: `work.bats` 14 (new: installed runs unset; the job's copy with args intact —
  spaces, empty, `-x` — and the job's `lib/anfs`, said once; a tool the job lacks runs installed
  with the job's lib; a job without core keeps the installed lib; a copy or checkout run by path
  never hands over; `nosuch`, `../x`, `.hidden` are 64 with the message, `ANFS_WORK_HOME=~/jobs`;
  nested: the second tool hands over, no second line, `ANFS_WORK_SAID` of another job says it
  again; a HOME reached through a symlink; `paths.sh`'s `ANFS_LIB_DIR` and `anfs_lib_dir` with and
  without a job and without core; the job's `paths.sh` is the one that ran; `anfs_lib_files ppm`:
  the job's `core.sh`, anfs-dev's installed `container.sh`, `new.sh` only in the job, a file the
  job deleted; under `/bin/bash` 3.2 with no newer bash for `bash.sh` the hand-over still happens;
  the entry-script checks above), `bash.bats` 12 (`work.sh` in the bootstrap set),
  `isolation.bats` 6 (new: a suite that sets HOME unsets ANFS_WORK or its helper does), ppm 67,
  psm 18.
- **Once, at the end**: `packages/anfs/tests/all`: first run **540 passed, 3 failed** (iso.bats 2:
  pmm run by path without `PMM_LIB_DIR` lost its beside-the-script fallback; settings.bats 1: the
  job variables not in the allow-list), fixed in `47fc72a`; then **543 passed, 0 failed**, 38 files
  (pmm and pcm off), 60 s.
- anfs-dev: `bats packages/dev/tests` **38/38** (new: `run` with `ANFS_WORK` exported runs the
  command without it or `ANFS_WORK_SAID`; `env` starts with the unset and, eval'd, takes it away;
  `show`'s last line).
- stack, `ANFS_TEST_LIB` = the job's core lib: git, obsidian, op, pikvm, remotefs, ruby, ssh,
  tailscale, ventoy: **216/221; the 5 failures are the known ones on main** (`ssh-hook.bats` 25, 31,
  34, 35 as numbered alone; `tsm.bats` 61). rpi-imager and firstmate have no bats suite.
- user: `bats packages/pjm/tests` with `PJM_TEST_ANFS_LIB` = the job's core lib: **13/14**; test 12
  ("dispatch sends only what is ready and tagged…") fails the same without this change (the
  script stashed, the job's lib and the live lib): not this plan's.
- Greps: `ANFS_LIB=\|\$HOME/\.local/lib/` in `packages/*/home/.local/bin` of core, stack, anfs-dev:
  only the preamble, the `paths.sh` check, the two literal `source` lines, and rpi-imager's last
  candidate. pjm keeps its `ANFS_LIB=` (preamble only, as planned): under a job it hands over to
  the job's pjm; that one sources the installed `paths.sh`, which hands over to the job's, but its
  `log.sh`, `cli.sh` and `table.sh` stay the installed ones (open question 2).
- **By hand, before merge**: a temp install under the job's TMPDIR (core, stack, anfs-dev copied
  from the job's trees into its sources, tools and `lib/{anfs,pcm}` linked as ppm links them), the
  real `~/work` as `ANFS_WORK_HOME`: `ANFS_WORK=config-03 wsm --version` → `anfs: running job
  config-03` / `wsm 0.8.0`; `anfs --version` the same; `anfs work show config-03` through the
  `anfs` seam: one line said, the last line the new one; `/bin/bash …/psm --version` hands over;
  `ANFS_WORK=nosuch pcm list` → 64 before anything runs; unset, plain. Cost: about 50 ms more per
  run under `ANFS_WORK` (the exec, the readlinks, `paths.sh` twice); nothing without it.

## Not run

- pmm's and pcm's suites are off (ROADMAP, 2026-10-19). pcm and pmm changed in their headers and
  library directory (`anfs_lib_dir`), pmm's `anfs` provisioner, and their helpers (the unset).
- Deferred to the end of the roadmap: `roundtrip debian` (Linux `readlink` forms, the preamble
  under Debian's bash), `roundtrip-macos`.
- Nothing was installed on this host; `ANFS_WORK` was never set against the real `HOME` (the live
  tools have no preamble before merge anyway).

## Deviations from the plan

- The entry-script checks live in `work.bats`, not `registry.bats` (above).
- pcm's and pmm's fallback to "libraries beside the script's real path" is gone (the plan's
  replacement implies it): a checkout run by path needs `PCM_LIB_DIR`/`PMM_LIB_DIR` or an
  installed `lib/pcm`/`lib/pmm`. Their helpers and pmm's e2e set them; core's `iso.bats` now does.
- Beyond the entry scripts, three library-side files moved off `$HOME/.local/lib/anfs`: ppm's
  `lib/ppm/sources.sh`, pmm's `anfs` provisioner, psm's and wsm's `install.sh` (so a job's ppm or
  pmm does not mix in the installed lib/anfs). pcm's edges and pmm's `sh` provisioner source only
  `bash.sh` and were left.
- The plan's "for the person after merge" lists a `ppm install -f` of every tool: not needed (see
  below): nothing new is linked.

## Shared files touched (for config-02 and config-04's rebase)

- core: `lib/anfs/paths.sh` (a block after the header comment, the `ANFS_LIB_DIR` line, a "Library
  directories" block before "Settings"; nothing else reformatted); the header of every entry
  script (anfs, ppm, pcm, pmm, psm, wsm: preamble, check, the `$ANFS_LIB` → `$ANFS_LIB_DIR`
  rewrites to the last `source`; anfs's `_need_lib` message and implode loop; pcm's and pmm's lib
  dir block); `AGENTS.md` (new section before "Development"; Lib Structure: two comment blocks,
  the rules paragraph, the floor sentence, the bootstrap set, ppm's library paragraph); tests: one
  top-level `unset` line in 19 bats files (registry.bats among them; conf.bats untouched), settings.bats's
  anfs allow-list line, iso.bats's setup, pcm/pmm/wsm `helper.bash`, `isolation.bats` (one test
  appended), `bash.bats` (bootstrap list).
- stack: every entry script's header; one `unset` line in 9 bats files and obsidian's helper.
- anfs-dev: `anfs-work` (header, `env`/`run`/`show`, epilogue), README, `work.bats`, `helper.bash`.
- user: `pjm`'s two lines.

## Open questions (for the person)

1. **user's `pjm` test helper** sets a temp `HOME` and does not unset `ANFS_WORK` (decision 12
   limits the user repo to the preamble line). `anfs work run` unsets it anyway; add the line there?
2. **pjm under a job gets the job's `paths.sh` but the installed `log.sh`/`cli.sh`/`table.sh`**,
   since it keeps `ANFS_LIB=` (the plan's choice). Move pjm's header to `$ANFS_LIB_DIR` too (four
   lines in the user repo)?
3. **pjm.bats test 12 fails on main** (dispatch, `status` 1 expected): not this plan's; worth a look.
4. Suggested bullet for ROADMAP "For an agent running this list" (yours to add): "**`ANFS_WORK`
   never in a test.** `ANFS_WORK=<job> <tool>` runs a job's code on live data, for a person trying a
   branch by hand. Agents do not use it (it changes real state); `anfs work run` unsets it."

## By hand, for the person after merge

Nothing is installed: `lib/anfs` is one directory link (so `work.sh` arrives with the pull) and
every entry script is already a link into its source (the new header arrives with the pull too).

1. `anfs src update` (core, stack, anfs-dev); the user source once its branch is merged and pulled.
2. Read-only checks, with any job that has core (this one until it is removed, or `anfs work new
   try core`):

   ```bash
   ANFS_WORK=config-03 wsm --version            # anfs: running job config-03, then wsm 0.8.0
   ANFS_WORK=config-03 /bin/bash -x ~/.local/bin/wsm --version 2>&1 | grep -m1 "exec "   # the exec into ~/work/config-03/core/...
   ANFS_WORK=config-03 ventoy help | head -3     # stack in the job: its ventoy; one line said
   ANFS_WORK=config-03 anfs work show config-03 | tail -1
   ANFS_WORK=nosuch pcm list; echo $?           # anfs: no job nosuch in ~/work (anfs work list), 64
   export ANFS_WORK=config-03; anfs work run config-03 core -- env | grep '^ANFS_WORK='; unset ANFS_WORK   # prints nothing
   ```

   With a job that changes a tool's output (its help, say), `ANFS_WORK=<job> <tool> help` shows
   the change.
