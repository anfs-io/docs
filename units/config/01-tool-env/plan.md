---
---

# Plan 01 — Each tool's `<tool>.env`, one loader, `anfs.conf` gone

Written 2026-10-21 against core main `9e8cb73` plus job `wsm-03` and linker plan 02 (both merged
first: unit decision 15)

## Context — read these files first

The job is `anfs work new <job> core stack anfs-dev`: core for the loader, the tools' files and
install.sh; stack for its tools' settings files; anfs-dev for its README's one line. Paths are
core's unless they name another repo.

- `units/config.md` — decisions 1, 6, 8, 10, 11; "Decided here" (where the files are, the loader, "default",
  `*_HOME` environment-only); the settings table; "Conflicts" (tsm, opm)
- `packages/anfs/home/.local/lib/anfs/paths.sh` — `ANFS_CONF`, `ANFS_LOCAL_CONF`,
  `anfs_tool_paths`, `anfs_load_conf` (called at source time)
- `packages/anfs/home/.config/anfs/anfs.conf` — every line moves (the unit's table)
- `install.sh` — `install_configs` (the `anfs.local.conf` seed), the header's file list
  (lines 23–44), `stow_base`'s comment on a layer's `anfs.conf`, `install_repo`'s comment
- The readers of each setting: `lib/anfs/sources.sh` (`ANFS_UPDATE_CACHE_DURATION`,
  `ANFS_QUIET_SKIPPED_REPOS`), `lib/anfs/skill.sh` (`ANFS_SKILLS_CLI_VERSION`, after wsm-03),
  `lib/ppm/platform.sh`, `bin/pcm` and `lib/pcm/{runtime,ingress,implode}.sh`, `bin/pmm` and
  `lib/pmm/{core,machine,build,image-test}.sh`, `bin/wsm` (`_spaces_home`, which expands `~`)
- `lib/ppm/platform.sh` — `HOMEBREW_UPDATE_CACHE_DURATION` (line ~98), renamed here
- stack: `packages/{remotefs,ventoy,pikvm,tailscale,op}/home/.local/bin/*`,
  `packages/obsidian/home/.local/bin/ovm`, `packages/tailscale/install.sh` (`TSM_AUTOJOIN`, the
  wrong `~/.config/ppm/ppm.local.conf`), `packages/tailscale/README.md`; tsm's `TSM_CONFIG_HOME`
  (lines ~38–40 and every use); opm's `OP_*` reads and `packages/op/install.sh`, its README and
  tests (`grep -rn 'OP_SHARED_VAULT\|OP_SA_CREDENTIALS_VAULT\|OP_PROVISION_TAG'` in stack)
- `AGENTS.md` — "Configuration: anfs.conf", "Key Files", "Lib Structure" (`paths.sh`), the
  "Repository Layout" sentence on `<dir>/<tool>`; `README.md` (line 123);
  `packages/{anfs,ppm,wsm}/README.md`; anfs-dev `packages/dev/README.md` (line 26)
- `packages/anfs/tests/bash.bats` (the parse list), `registry.bats`

## Overview

Each tool ships its settings, documented, in its own config directory, and reads only those and
anfs's. A machine's overrides sit beside them. Nothing writes a settings file as a side effect.

## Implementation

### 1. The loader (`paths.sh`)

- `ANFS_CONF`/`ANFS_LOCAL_CONF` go. `anfs_conf_files <tool>` prints the two paths for a tool:
  `${<TOOL>_CONFIG_HOME:-$XDG_CONFIG_HOME/<tool>}/<tool>.local.env`, then `<tool>.env` (anfs:
  `$ANFS_CONFIG_HOME/anfs.{local.,}env`). Used by the loader and by plan 02.
- `anfs_conf_read <file>`: the parser, as today (`NAME=value`, comments and blanks skipped, one
  pair of matching quotes stripped, no expansion; a leading `export ` is accepted, as pcm's own
  parser does, so plan 04 can use this one), printing `NAME<TAB>value` lines. No export.
  Plan 02 reads files through it; the loader exports from it.
- `anfs_load_conf [tool]`: no tool is anfs. For each of the two files, each name not yet set is
  exported. Skipped, with one warning per file and name (`anfs: <file>: <NAME> is read from the
  environment only`): `ANFS_WORK`, and any `*_HOME` that `paths.sh` or `anfs_tool_paths` sets
  (`ANFS_{CONFIG,DATA,STATE,CACHE}_HOME`, `<TOOL>_{CONFIG,DATA,STATE,CACHE}_HOME`).
- `paths.sh` calls `anfs_load_conf` at source time (anfs's files, every tool); `anfs_tool_paths
  <tool>` ends with `anfs_load_conf <tool>`. A tool calls it once; a second call loads nothing
  new (names already set).
- The header's file list: `~/.config/<tool>/<tool>{,.local}.env` in place of `anfs{,.local}.conf`.

### 2. The shipped files

Each is `packages/<pkg>/home/.config/<tool>/<tool>.env`, linked per file by ppm like any file. The
header (same words in each): what the file is, that `<tool>.local.env` beside it is this
machine's, that the environment wins, the format, and `<tool> config` (plan 02; until it merges,
the file). Then every setting the tool reads: one comment line saying what it does, then
`#NAME=default` (or `NAME=value` where the shipped value is not the code's default: the two
`HOMEBREW_NO_*`, `ANFS_UPDATE_CACHE_DURATION` and `PPM_BREW_UPDATE_CACHE_DURATION` stay active, as
today).

- core `anfs/…/anfs/anfs.env`: `ANFS_UPDATE_CACHE_DURATION`, `ANFS_QUIET_SKIPPED_REPOS`,
  `ANFS_SKILLS_CLI_VERSION`, `HOMEBREW_NO_AUTO_UPDATE`, `HOMEBREW_NO_ENV_HINTS`.
- core `ppm/…/ppm/ppm.env`: `PPM_BREW_UPDATE_CACHE_DURATION` (decision 8): `platform.sh` reads
  `${PPM_BREW_UPDATE_CACHE_DURATION:-86400}`; `HOMEBREW_UPDATE_CACHE_DURATION` is read nowhere
  after (no shim). install.sh's bootstrap sources `platform.sh` without settings and gets the
  default, as today.
- core `pcm/…/pcm/pcm.env`: the seven `PCM_*` of the table.
- core `pmm/…/pmm/pmm.env`: the five `PMM_*`.
- core `wsm/…/wsm/wsm.env`: `WSM_SPACES_HOME` (wsm keeps expanding `~` itself).
- core psm: none (its one setting is anfs's `ANFS_SKILLS_CLI_VERSION`). The registry test below
  expects none.
- `packages/anfs/home/.config/anfs/anfs.conf` is deleted.
- stack (the tools that call `anfs_tool_paths`, plus tsm and opm, which get the call):
  `remotefs.env` (`REMOTEFS_CONNECT_TIMEOUT`, `REMOTEFS_MOUNT_TIMEOUT`, `REMOTEFS_STAT_TIMEOUT`,
  `REMOTEFS_SSHFS`, `REMOTEFS_DEFAULT_OPTIONS`; `REMOTEFS_CONFIG` is a path, environment-only),
  `ventoy.env` (`VENTOY_ISO_DIR`, `VENTOY_IMAGE_SIZE`; `VENTOY_DISK` is per run, not a setting —
  check its use and say in the log), `pikvm.env` (`PIKVM_HOST`, `PIKVM_SSH_OPTS`,
  `PIKVM_MSD_IMAGE`), `ovm.env` (`OVM_PROBE_SECONDS`), `tsm.env` (`TSM_AUTHKEY_MODE`,
  `TSM_AUTOJOIN`), `opm.env` (`OPM_SHARED_VAULT`, `OPM_SA_CREDENTIALS_VAULT`,
  `OPM_PROVISION_TAG`, `OPM_TAG`: opm's `OP_*` are renamed here, decision 10, no shim, in its
  code, hook, tests and README). Each tool's defaults are read from its code; a
  variable the code reads that is a test hook or an internal switch (`*_IN_FNOX`, `*_LIB`,
  `*_BIN`, `*_BASH_REEXEC`) is not listed.
- pikvm, opm and tsm do not call `anfs_tool_paths` today: they gain `anfs_tool_paths <tool>` after
  `paths.sh` (which also gives them `<TOOL>_*_HOME`). tsm used `TSM_CONFIG_HOME` for tailscale's
  directory (`~/.config/tailscale`); that variable becomes `TSM_TAILSCALE_HOME` (same default,
  environment-only, a path), so `TSM_CONFIG_HOME` is tsm's own (`~/.config/tsm`, where `tsm.env`
  is) and plan 02's library needs no special case. No shim; `TSM_CONFIG` (the file) keeps its name
  and now defaults under `TSM_TAILSCALE_HOME`.
- stack `tailscale/install.sh`: the hook calls `anfs_load_conf tsm` before reading
  `TSM_AUTOJOIN` (hooks run inside ppm, which loads anfs's and ppm's only); its messages and the
  README name `~/.config/tsm/tsm.local.env`, not `~/.config/ppm/ppm.local.conf`.

### 3. install.sh and ppm

- `install_configs` seeds `user.list` only (an empty list is the source list's file, which the
  tools need; the seed is unchanged). No settings file is written (decision 5 of the unit: no
  CONFIG as a side effect).
- `PPM_GROUP_ID` is already gone (linker 02). `grep -rn PPM_GROUP_ID` in core finds nothing; if
  it does, stop: linker 02 is not merged.
- The header and comments: `~/.config/anfs/anfs.env` (linked from core/anfs) and
  `anfs.local.env` (yours, by hand) in the file list; a layer's `anfs.env` in `stow_base`'s
  comment.

### 4. Documentation

Core `AGENTS.md`: "Configuration: anfs.conf" becomes "Configuration: `<tool>.env`" (decision 1 and
"Decided here": the files, precedence, the format, environment-only names, a hook calling
`anfs_load_conf <tool>`, `<tool> config` from plan 02 named as coming); "Key Files"; "Lib Structure"
(`paths.sh`: `anfs_load_conf [tool]`, `anfs_conf_read`, `anfs_conf_files`); "Package Structure"
(a tool ships `home/.config/<tool>/<tool>.env` documenting every setting). The examples that name
`PSM_CLI_VERSION` (wsm-03 renamed it) or `anfs.conf`. `README.md`, `packages/anfs/README.md`
(the table's `anfs.conf` row; the layer paragraph), `packages/ppm/README.md`,
`packages/wsm/README.md` (`wsm.local.env`). stack's READMEs of the tools above (one line each:
the file and its settings). anfs-dev `packages/dev/README.md`: `$ANFS_WORK_HOME` is read from the
environment only (decision 11), as is `ANFS_WORK_TMP`.

## Test Spec

- `packages/anfs/tests/conf.bats` (new; temp `HOME` and XDG dirs): `anfs_conf_read` on quoted,
  unquoted, blank, comment, `export NAME=` and malformed lines; `anfs_load_conf` with no tool loads
  `anfs.local.env` over `anfs.env`; the environment beats both; `anfs_load_conf pcm` reads
  `$PCM_CONFIG_HOME/pcm{,.local}.env` and a `PCM_CONFIG_HOME` pointed elsewhere moves them; a tool
  whose directory does not exist loads nothing and fails nothing; `ANFS_WORK=x` and
  `PCM_CONFIG_HOME=/x` in a file are not exported and warn once; `anfs.conf` in the directory is
  not read.
- The same file: sourcing `paths.sh` then `anfs_tool_paths pcm` in a home with only `anfs.env`
  installed leaves every `PCM_*` setting unset (the encapsulation).
- `registry.bats` (or a new `settings.bats`): for each core tool with a shipped file, every
  `: "${<TOOL>_X:=…}"` and `${<TOOL>_X:-…}` in its bin and lib that is not a `*_HOME`, a `*_DIR`
  path, a test hook or in an allow-list in the test has a `#<TOOL>_X=` or `<TOOL>_X=` line in its
  `.env`, and each line names a variable the code reads. psm has no file and no setting.
- `bash.bats`: nothing new to parse (no new library file); `paths.sh` still parses.
- `grep -rn 'anfs\.conf\|anfs\.local\.conf\|ANFS_LOCAL_CONF\|ANFS_CONF\b'` in core and anfs-dev
  finds nothing outside `tests/` history; in stack, nothing.
- `grep -rn 'HOMEBREW_UPDATE_CACHE_DURATION'` in core finds nothing; ppm's brew-update test (or a
  new one) uses `PPM_BREW_UPDATE_CACHE_DURATION`.
- stack: each tool's suite passes; tailscale's hook test (if any) reads `TSM_AUTOJOIN` from a
  temp `tsm.local.env`; tsm finds tailscale's config under `TSM_TAILSCALE_HOME`;
  `grep -rn 'OP_SHARED_VAULT\|OP_SA_CREDENTIALS_VAULT\|OP_PROVISION_TAG\|TSM_CONFIG_HOME.*tailscale'`
  in stack finds nothing.

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing that boots
a VM or a box runs, and nothing is installed on this host. pmm's and pcm's suites stay off
(`ANFS_TESTS_PMM_PCM` unset); the log says they were not run.

- [ ] While working, the file you touch: `bats packages/anfs/tests/conf.bats`,
      `bats packages/anfs/tests/registry.bats`
- [ ] `bats packages/anfs/tests`, `bats packages/ppm/tests`, `bats packages/wsm/tests`,
      `bats packages/psm/tests`
- [ ] stack: `anfs work run <job> stack -- env ANFS_TEST_LIB=<job's core>/packages/anfs/home/.local/lib/anfs
      bats packages/{remotefs,ventoy,pikvm,obsidian,tailscale,op}/tests` (those that have tests)
- [ ] By hand with `HOME` and XDG dirs in a temp dir and the job's core linked there: `pcm help`
      and `pmm help` run; `env -i HOME=… bash -c 'source …/paths.sh; anfs_tool_paths pcm; env' `
      shows `PCM_HEALTH_TIMEOUT` only when `pcm.env` has it active
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`), by hand: `ppm install -f core/anfs core/ppm
  core/pcm core/pmm core/wsm` (links each `<tool>.env`); move this Mac's
  `~/.config/anfs/anfs.local.conf` lines: `PCM_MACHINE_MEMORY=…` to
  `~/.config/pcm/pcm.local.env`, `PPM_GROUP_ID` dropped (linker 02 removed it); then delete
  `anfs.local.conf` and the dangling `~/.config/anfs/anfs.conf` link. user/anfs ships an
  `anfs.conf` (`ANFS_QUIET_SKIPPED_REPOS=true`) and rws/anfs an older copy: in your own repos,
  delete them and put the one line in `~/.config/anfs/anfs.local.env` (or ship it as a layer's
  `anfs.env`, the whole file). stack tools: `ppm install -f stack/<tool>` for each installed one; if you set `OP_*` for opm or
  `TSM_CONFIG_HOME` for tsm anywhere (your shell, fnox), rename them to `OPM_*` and
  `TSM_TAILSCALE_HOME`.
- Deferred to the end of the roadmap (named in `log.md`): `roundtrip debian`, `roundtrip-macos`
  (a fresh install.sh: no `anfs.local.conf` written, `anfs.env` linked).
