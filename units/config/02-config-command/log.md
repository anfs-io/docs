---
plan: config/02-config-command
job: config-02
date: 2026-10-22
---

# Log: config/02-config-command

## The plan against the code

Written against core `9e8cb73` plus plan 01; the job branched from core `533935e`, stack
`2f989ee` (plan 01 merged). Plan 01's API was there as its log says (`anfs_conf_files`,
`anfs_conf_read` as written, `anfs_conf_expand`, `anfs_conf_env_only`, `anfs_load_conf`). The one
thing the plan asked of `paths.sh` (`ANFS_CONF_LOADED`) was not there yet; added here. Nothing in
the plan was out of date.

## What was built

- **`lib/anfs/config.sh`** (new, core): `anfs_config_declare` declares `config list [--export]`
  (anfs: `--all` too), `config get <setting>`, `config set <assignment>...`, `config unset
  <setting>...` for `$CLI_TOOL`; the `cmd_config_*` functions; `anfs_config_rows <tool>`;
  `complete_setting` (get, unset) and `complete_assignment` (set: `NAME=`), both a read of the
  shipped file in bash, no fork. The group's one-line summary: `Settings in ~/.config/<t>:
  <t>.local.env over <t>.env (shipped), the environment over both`.
  - Declared names: the shipped file's active and `#NAME=value` lines (no space after `#`); an
    active line beats a commented one of the same name. A name only in `.local.env` is listed
    FROM `local`; `get`/`set` refuse it. Environment-only names in a file are never rows.
  - FROM: `env` (exported, and not in `ANFS_CONF_LOADED`), `local`, `shipped`, `default`. `file`:
    the local file, the shipped file (for `shipped` and `default`), null for `env`.
  - `ANFS_WORK` is a row of anfs's list (FROM `env`) when set; a line in `anfs.local.env` gives
    none (the loader's warning still prints).
  - `list --all`: anfs, then each `$XDG_CONFIG_HOME/<d>/<d>.env`, plus a tool whose
    `<TOOL>_CONFIG_HOME` is exported and holds `<tool>.env` (a `_` in the variable tried as `-`
    too); sorted; leading `TOOL` / key `tool`.
  - `set`: refuses a missing `=`, a malformed name, `ANFS_WORK`, any `anfs_conf_env_only` name,
    an undeclared one, a value with `'` or a newline (all 64, `<tool> config: ...`), and checks
    every argument before writing anything. Replaces the first active line (with or without
    `export `) in place, drops later lines of the name, appends otherwise; a missing file gets
    `# <tool> settings for this machine (<tool> config set and unset write here; the defaults are
    in <tool>.env)`. Temp file beside the target, then `mv`. One warning per name the
    environment also sets. Prints `NAME=value (~/.config/<tool>/<tool>.local.env)`, or
    `anfs_result name value file` per name with `--json`.
  - `unset`: removes every active line of each name; a name not there is a note, exit 0; no file:
    nothing created. `--json`: `{name, removed, file}` per name.
- **`paths.sh`** (shared file): `anfs_load_conf` appends each name it exports to
  `ANFS_CONF_LOADED`; set to empty (when unset) and `export -n` before anfs's own load at the end
  of the file; the Settings header names it. 5 lines.
- **Core tools**: anfs, ppm, pcm, pmm, wsm source `config.sh` and call `anfs_config_declare`
  before `cli_standard` (pcm and pmm in `lib/<tool>/commands.sh`, with `config.sh` sourced in
  their entry scripts and in their test helpers). `config` skips the source list / catalogue:
  ppm's no-source case, pcm's `load_sources` skip, pmm's catalogue skip.
- **Stack tools** (ovm, tsm, ventoy, remotefs, pikvm, opm): `config.sh` sourced when present,
  `! declare -F anfs_config_declare >/dev/null || anfs_config_declare` before `cli_standard`;
  their "out of date" checks unchanged. ovm: `config` is added to the commands that do not
  register the vault you stand in.
- **Shipped `.env` headers** (5 core, 6 stack): "`<tool> config list` shows them and where each
  is from; `<tool> config set NAME=value` writes <tool>.local.env." (paragraph rewrapped).
- **Docs**: core AGENTS.md "Configuration" (the group, a table of the verbs, FROM, declared
  names only, refusals, `ANFS_WORK`, not in `RPC_GROUPS`; the API bullet adds
  `ANFS_CONF_LOADED`), "Lists and --json" (rule 5 names `<tool> config list`; a key row), "The
  Command Line" (a tool with settings calls `anfs_config_declare`; iff test), "Lib Structure"
  (`config.sh`, `ANFS_CONF_LOADED`). READMEs one line each: anfs, ppm, pcm, wsm (core; pmm has no
  README); obsidian, tailscale, remotefs, ventoy, op (stack; pikvm has none). stack AGENTS.md:
  the stack tools' pattern.
- `config` is not in `RPC_GROUPS` (through `anfs rpc` it is a write).

## Commits (branch config-02, not pushed)

core:
- `98dd8e0` config: `lib/anfs/config.sh`, the group in anfs, ppm, pcm, pmm, wsm;
  `ANFS_CONF_LOADED`; headers; READMEs; `config.bats`, registry, bash, conf tests
- `4f6ccf5` docs: AGENTS.md

stack:
- `0f52fca` ovm, tsm, ventoy, remotefs, pikvm, opm: the group; headers; a test per suite;
  READMEs, AGENTS.md; opm README's example

Versions: the pre-commit hook bumped nothing: core's anfs (0.2.15), ppm (0.2.8), pcm (0.1.10),
pmm (0.1.5), wsm (0.1.9) and stack's obsidian (0.2.1), op (0.1.5), pikvm (0.1.2), remotefs
(0.1.1), tailscale (0.1.3), ventoy (0.1.3) already carry their bump for the unpushed series (one
bump per series). `WSM_VERSION` (0.8.0) and `OVM_VERSION` (0.2.1) likewise already bumped in it.

## Decisions made during the work

- **VALUE is the effective value**: a file's value with its `~` expanded (`anfs_conf_expand`),
  for `list`, `get` and `--export`, so `get` gives what the tool gets and `--export` is usable
  as is; on a terminal the table shows it as `~/…` anyway. `set` writes the value as typed (`~/d`
  stays `~/d` in the file).
- **`get` of an empty value**: exit 1 and nothing only for an empty `default` (the code's default
  is "none"); an explicit `NAME=` in `.local.env` or an empty exported value prints an empty line,
  exit 0.
- **`list --export` leaves out empty defaults** besides `env` rows (deviation, small): exporting
  `''` for a `${X-default}` reader (pcm's `PCM_ATTACHED_SERVICES-postgres valkey`) would change
  behaviour; leaving it out is the same as the tool's default.
- **Quoting**: a value is single-quoted unless it is only `[A-Za-z0-9_./:@%+,=~-]` (broader than
  the plan's "space, quote or #"; the reader strips the pair either way).
- **`unset`** takes a declared name or any name with a line in `.local.env` (a stray one, or an
  environment-only line the loader warns about): that is how such a line goes.
- **`set` through a link**: when `.local.env` is a symlink (a person's dotfiles), its target is
  written and the link kept. An existing file keeps its mode (`cp -p` to the temp file first); a
  new one gets the umask's.
- **anfs and ppm source `config.sh` when present** (deviation: the plan said only stack's tools
  do): both already load every lib/anfs file but `paths.sh`, `sources.sh`, `cli.sh` that way,
  because they are what repairs an install whose `lib/anfs` predates them. pcm, pmm, wsm require it.
- **`anfs config get ANFS_WORK`** prints the job when set, 1 when not (the row is anfs's).
- **Environment detection** checks the exported attribute (`${!name@a}`), not only "set": a tool
  that assigns a default without exporting (`: "${PMM_LIBVIRT_URI:=…}"`) is not `env`.
- `config.bats` links the shipped files into the temp `~/.config`, as ppm does, instead of
  copying them; the fake tool's file is written by the test.

## Tests (all through `anfs work run config-02 …`, `</dev/null`)

- While working: `config.bats` 17 (new: every item of the plan's spec but the stack ones),
  `registry.bats` 42 (new: a core tool ships `<tool>.env` iff it declares the group; `NAME VALUE
  FROM` and `TOOL NAME VALUE FROM` in rule 5's list), `bash.bats` 10 (`config.sh` asserted in the
  parse list), `conf.bats` 16 (new: `ANFS_CONF_LOADED` holds what the loader exported, not the
  environment's, and is not exported), `settings.bats` 8, `rpc.bats` 21.
- **Once, at the end**: `packages/anfs/tests/all`: **547 passed, 0 failed**, 38 files (pmm and
  pcm off), 68 s.
- stack, `ANFS_TEST_LIB` = the job's core lib (new: one `config list`/`set`/`get` test per
  suite): obsidian 60/60, tailscale 29/30, ventoy 19/19, remotefs 18/18, pikvm 19/19, op 35/39.
  **The 5 failures are the known ones on main**: `tsm.bats` "an option the command does not take
  is a usage error" (25 in this run) and `ssh-hook.bats` 1, 7, 10, 11 (28, 34, 37, 38 in this run).
- By hand, a temp `HOME` under the job's `TMPDIR` with the job's anfs, ppm, pcm, pmm, wsm linked,
  `env -i`: `pcm config list/set/get/unset/--export/--json`, `PCM_HEALTH_TIMEOUT=5` FROM env,
  `anfs config list --all` (anfs, pcm, pmm, ppm, wsm), `anfs container config list`, `ppm
  config` with no source list, `pmm config list`, `wsm config list`, every refusal (64),
  completion and help.

## Not run

- pmm's and pcm's suites are off (ROADMAP, 2026-10-19) and were not run. pcm: `commands.sh` (one
  line), the entry script (source + the `load_sources` skip), `pcm.env`'s header, README, test
  helper; pmm: `commands.sh` (one line), entry script (source + the catalogue skip), `pmm.env`'s
  header, test helper. `bash -n` on each passed. **shellcheck is not installed on this host**, so
  it was not run on any file.
- anfs-dev: untouched.
- Deferred to the end of the roadmap: none (no VM or box behaviour changes).

## Shared files touched (for the rebase with config-03 and config-04)

- core `packages/anfs/home/.local/lib/anfs/paths.sh`: the Settings header (2 comment lines),
  one line in `anfs_load_conf`, two lines before the final `anfs_load_conf` call. No lib-root
  change, nothing reformatted.
- core `AGENTS.md`: "Configuration: `<tool>.env`" (the plan-02 bullets and a new block at its
  end), "Lists and --json" (rule 5, one key row before `pmm ps`), "The Command Line" (one bullet
  after `cli_section`), "Lib Structure" (`paths.sh` comment, `config.sh` entry).
- Entry scripts, mid-file only (none of their headers, so config-03's preamble does not
  collide): core `anfs`, `ppm`, `pcm`, `pmm`, `wsm`; stack `ovm`, `tsm`, `ventoy`, `remotefs`,
  `pikvm`, `opm`. Plus `lib/pcm/commands.sh`, `lib/pmm/commands.sh` (one line each).
- `packages/pcm/tests/helper.bash` (two lines after `source cli.sh`, a few lines below
  `PCM_ENV_HOME`'s export, which config-04 removes), `packages/pcm/README.md` (the opening
  paragraph; config-04 rewrites "Local overrides" below it), `pcm.env`'s header.
- Every shipped `.env` header (5 core, 6 stack); stack `AGENTS.md` ("Tools shipped by packages").

## Open questions

None blocking. One for the person:

1. `anfs config list --all` finds a tool by `$XDG_CONFIG_HOME/<d>/<d>.env`. A tool whose
   directory holds a stray `<d>.env` that is not an anfs settings file would be listed too
   (nothing seen on this Mac's layout; none of `~/.config` was read beyond what the plan names).

## By hand, for the person after merge

After plan 01's steps (its `log.md`):

1. `anfs src update` (`lib/anfs` is one link: `config.sh` is live at once; the tools' new
   declarations come with the pull, as their scripts are linked from the clones). Then, as the
   plan asks (this plan adds no linked file, so it is a check):
   `ppm install -f core/anfs core/ppm core/pcm core/pmm core/wsm`, and each installed stack tool
   among obsidian, tailscale, ventoy, remotefs, pikvm, op (`ppm install -f stack/<name>`).
2. Check, nothing written:

   ```bash
   pcm config list                  # PCM_MACHINE_MEMORY 4096 local, once plan 01's step 3 is done
   anfs container config list       # the same
   anfs config list --all           # anfs, then ppm, pcm, pmm, wsm and the installed stack tools
   ```

3. Set, read and take back one value:

   ```bash
   pcm config set PCM_HEALTH_TIMEOUT=300 && pcm config get PCM_HEALTH_TIMEOUT
   pcm config unset PCM_HEALTH_TIMEOUT && pcm config list | grep HEALTH   # default again
   ```

4. Run pcm's and pmm's suites: `ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm pmm`.
5. Completion: `zcomp` caches nothing about commands, so `pcm config get <TAB>` works as is.
