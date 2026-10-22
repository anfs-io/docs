---
---

# Plan 02 — `<tool> config`: one command group from `lib/anfs/config.sh`, in every tool with settings

Written 2026-10-21 against core main `9e8cb73` plus job `wsm-03` and linker plan 02, and plan 01
of this unit (merged first). Independent of plans 03 and 04. Replaces the draft's `02-anfs-config`
(one `anfs config` over every tool), revised after review the same day.

## Context — read these files first

The job is `anfs work new <job> core stack`: core for the library and its tools; stack for its
tools on `cli.sh`. Paths are core's unless they name another repo.

- `units/config.md` — decisions 1, 2 (as revised), 3 (`anfs config list` shows `ANFS_WORK`);
  "Decided here" ("default", what a tool declares, which tools declare the group, which tools
  `--all` shows); plan 01's `log.md`
- `packages/anfs/home/.local/lib/anfs/paths.sh` after plan 01 — `anfs_conf_files`,
  `anfs_conf_read`, `anfs_load_conf`
- `packages/anfs/home/.local/lib/anfs/cli.sh` — `cli_group`, `cli_cmd "<group> <sub>"`,
  `--opt`/`--arg`, `cli_standard`, `CLI_TOOL`, how `cmd_<group>_<sub>` is dispatched and how
  completion kinds (`complete_<kind>`) are asked
- `packages/anfs/home/.local/lib/anfs/table.sh` — `anfs_row`, `anfs_table`, `anfs_result`,
  `anfs_json_on`; `AGENTS.md` "Lists and --json" (rule 5) and "The Command Line"
- `packages/anfs/home/.local/bin/anfs` — the `anfs <resource>` seam (`anfs container config …`
  execs `pcm config …`), `RPC_GROUPS`
- Each tool's command declarations: core `packages/{anfs,ppm,pcm,pmm,wsm}/home/.local/bin/*`
  (pcm's are in `lib/pcm/commands.sh`, pmm's in `lib/pmm/commands.sh`); stack
  `packages/{obsidian/…/ovm,tailscale/…/tsm,ventoy/…/ventoy,remotefs/…/remotefs,pikvm/…/pikvm,op/…/opm}`
- `packages/anfs/tests/registry.bats`, `tests/bash.bats` (the parse list), `tests/rpc.bats`

## Overview

Each tool answers for its own settings: `pcm config list` shows pcm's, where each value came
from, and `pcm config set` writes pcm's `.local.env`. The code is one library; the command is the
same in every tool. `anfs config` is anfs's own settings, and `anfs config list --all` is the one
view over every installed tool.

## Implementation

### 1. `lib/anfs/config.sh`

Functions only, the bash floor. Needs `paths.sh` (plan 01's reader and file paths), `table.sh`,
`cli.sh`.

- `anfs_config_declare`: declares, for `$CLI_TOOL`, the group and its commands (called by a tool
  among its declarations, before `cli_standard`):

  ```
  <tool> config list [--export]     NAME VALUE FROM (--json: name, value, from, file)
  <tool> config get NAME            the effective value; 1, nothing printed, when unset
  <tool> config set NAME=VALUE...   into this tool's .local.env
  <tool> config unset NAME...       from this tool's .local.env
  ```

  anfs's `list` also takes `--all`. The `cmd_config_*` functions are defined by the library;
  they read the tool from `$CLI_TOOL` (`anfs config` is anfs's).
- `anfs_config_rows <tool>`: one row per setting the tool declares. **Declared** names are those
  of its shipped file, active or commented `#NAME=value` (no space after `#`; prose comments are
  not settings), then any only in its `.local.env` (listed, FROM `local`, so a stray line is
  visible; `get`/`set` still refuse it). **Value and FROM**, first that applies: `env` (in the
  tool's inherited environment: see below), `local`, `shipped` (an active line), `default` (the
  commented line's value). Row keys: `name`, `value`, `from`, `file` (the local or shipped file;
  the shipped one for `default`; null for `env`).
- **The inherited environment**: plan 01 exports the files' values into the tool's own process,
  so "set in the environment" must be known before the loader runs. `paths.sh`'s `anfs_load_conf`
  records the names it exported (`ANFS_CONF_LOADED`, a space-separated list, not exported to
  children); a name set in the environment and not in that list is FROM `env`. A tool run by
  another tool inherits the parent's exported settings as environment, and `env` is then true.
- `ANFS_WORK`, when set: a row in `anfs config list` (`ANFS_WORK <job> env`), never in a file
  (decision 3).
- **list --all** (anfs only): the tools of "Decided here" (anfs, then every `$XDG_CONFIG_HOME/<d>/<d>.env`,
  a `<D>_CONFIG_HOME` in the environment honoured), each through `anfs_config_rows`, in this one
  process; a leading `TOOL` column (key `tool`). Reading another tool's files for display is
  read-only and needs nothing of that tool's code.
- **get NAME**: refuses a name the tool does not declare (64, `pcm config: PCM_X is not a pcm
  setting (pcm config list)`). Prints the value `list` shows.
- **set NAME=VALUE...**: the same refusal; also refuses `ANFS_WORK` and the environment-only
  `*_HOME` names (64, the reason), and a malformed name. Writes the tool's `.local.env` (plan 01's
  `anfs_conf_files`): replaces the `NAME=` line in place or appends; creates the file with a
  one-line header when missing. A value with a space, quote or `#` is single-quoted; a value with
  a single quote is refused (64: the parser strips one pair and has no escapes). Atomic (temp file
  in the same directory, then `mv`). When the name is also in the inherited environment: one line
  saying the environment wins. `anfs_result name value file` for `--json`.
- **unset NAME...**: removes the line from `.local.env` (never the shipped file, which is the
  package's); absent: a note, exit 0.
- **list --export**: `export NAME='value'` for each row whose FROM is not `env`, for `eval
  "$(pcm config list --export)"`. With `--all`, every tool's.
- **Completion**: `get`, `set`, `unset` complete the tool's declared names (reads the files, no
  subprocess); a `complete_setting` kind in the library.
- **Help**: the group's summary names the two files and the precedence in one line.

### 2. The tools

- Each tool that ships a `<tool>.env` (unit "Decided here") sources `$ANFS_LIB_DIR/config.sh`
  with its other libraries and calls `anfs_config_declare` among its declarations: core's anfs,
  ppm, pcm, pmm, wsm; stack's ovm, tsm, ventoy, remotefs, pikvm, opm. psm, anfs-dev's tools and
  stack's `ssh-key`, `gem-local`, `git-account` do not (no settings).
- stack's tools load `config.sh` only when it exists (`[[ -f … ]]`), so a stack tool runs against
  a core that predates this plan, without the group (their existing "out of date" checks name the
  files they require; `config.sh` is not added to them).
- tsm: nothing special; plan 01 made `TSM_CONFIG_HOME` tsm's own (`~/.config/tsm`), tailscale's
  directory being `TSM_TAILSCALE_HOME`.
- `anfs <resource> config …` needs nothing: the seam execs the tool with the rest of the line.
- `config` is not added to `RPC_GROUPS` (settings through the API is `toolkit-api`'s question).

### 3. Documentation

Core `AGENTS.md`: "Configuration" (the group, the verbs, FROM, declared names only, `anfs config
list --all`); "Lists and --json" (`<tool> config list`: `name`, `value`, `from`, `file`; with
`--all` a leading `tool`; rule 5, FROM is not a state); "The Command Line" (a tool with settings
calls `anfs_config_declare`); "Lib Structure" (`config.sh`). The shipped files' headers (plan 01)
name `<tool> config` in place of editing the file. Each tool's README: one line.

## Test Spec

`packages/anfs/tests/config.bats` (new; temp `HOME` and XDG dirs; a fake tool script on `cli.sh`
that declares the group, with a shipped `fake.env`; and the job's `pcm`/`anfs` scripts with their
`.env` copied into the temp config dirs):

- `list`: one row per declared setting; FROM `default` for a commented line, `shipped` for an
  active one, `local` for a `.local.env` line, `env` for a name exported before the tool runs; a
  name only in `.local.env` is listed FROM `local`.
- `--json` keys `name value from file`; `file` null for `env`.
- `set FAKE_X=300` writes `~/.config/fake/fake.local.env` (created, header); a second `set`
  replaces the line; a value with a space is quoted and reads back whole through `get`; `set
  OTHER_X=1` (not declared) 64; `set ANFS_WORK=x` 64; `set FAKE_CONFIG_HOME=/x` 64; a single quote
  64; with `FAKE_X` in the environment, `set` warns and `get` prints the environment's.
- `unset` removes the line and keeps the others; an absent name exits 0.
- `list --export` output, `eval`'d in an `env -i` bash, sets the values; `env` rows are left out.
- anfs: `anfs config list` has `ANFS_*` and `HOMEBREW_NO_*` and no `PCM_*`; with `ANFS_WORK=j` a row
  FROM `env`; `ANFS_WORK=x` in `anfs.local.env` gives no row; `anfs config list --all` has a
  `tool` column and pcm's rows when `pcm.env` is in the temp config dir, none when it is not.
- `pcm config set PMM_INSTALL=x` is refused; `anfs container config list` equals `pcm config list`
  (through the seam).
- `registry.bats`: a core tool ships `home/.config/<tool>/<tool>.env` if and only if it calls
  `anfs_config_declare`; the header `NAME VALUE FROM` (and `TOOL NAME VALUE FROM`) passes rules
  2 and 5.
- `bash.bats`: `config.sh` in the parse list.
- stack: each of the six tools' suites passes; one test per tool (or one shared in the log) that
  `<tool> config list` runs on a temp home.

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing that boots
a VM or a box runs, and nothing is installed on this host. pmm's and pcm's suites stay off (their
command declarations change; `bash -n` and `shellcheck` on their files); the log says so.

- [ ] While working, the file you touch: `bats packages/anfs/tests/config.bats`,
      `bats packages/anfs/tests/{registry,bash}.bats`
- [ ] `bats packages/anfs/tests`, `bats packages/ppm/tests`, `bats packages/wsm/tests`
- [ ] stack: `anfs work run <job> stack -- env ANFS_TEST_LIB=<job's core>/packages/anfs/home/.local/lib/anfs
      bats packages/{obsidian,tailscale,ventoy,remotefs,pikvm,op}/tests` (those that have tests)
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`): `ppm install -f core/anfs core/ppm core/pcm
  core/pmm core/wsm` and the six stack tools installed here; `pcm config list` (this Mac's
  `PCM_MACHINE_MEMORY` FROM `local`, once plan 01's by-hand move is done); `pcm config set
  PCM_HEALTH_TIMEOUT=300`, `get`, `unset`; `anfs container config list`; `anfs config list --all`.
  Run pcm's and pmm's suites (`ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm pmm`).
- Deferred to the end of the roadmap (named in `log.md`): none (no box or VM behaviour changes).
