---
objective: Each tool owns its settings — a shipped `<tool>.env` and a machine's `<tool>.local.env` beside its other config, read through one loader and one `config` command in every tool; `ANFS_WORK=<job>` runs a job's code on live data; pcm's per-container settings sit beside the definition; the XDG rule is written down and held to
status: plans 01–05 written 2026-10-21; decisions taken with the person 2026-10-21, revised after review the same day
---

One file, `~/.config/anfs/anfs.conf`, holds every tool's settings. It is shipped by `core/anfs`,
so a machine whose `system` stack installs only some tools still gets pmm's settings, and a tool
cannot ship or document its own. Overrides go in `anfs.local.conf`, which install.sh seeds as a
side effect. pcm has a second settings scheme of its own (`~/.config/pcm/env/<name>.env`). Nothing
lists what is set, where it came from, or lets a person set one without opening a file. And there
is no way to run a job's code (`~/work/<job>/`) against the live install's data short of copying
files around.

This unit gives each tool its own settings file, one loader, and a `config` command group in
every tool (`pcm config set …`, `anfs config list --all`); adds
`ANFS_WORK=<job>`; moves pcm's per-container settings beside the definitions; and writes the XDG
rule (what goes in CONFIG, STATE, DATA, CACHE, RUNTIME) into core `AGENTS.md`, with an audit of
every write into a config directory.

Written against core main at `9e8cb73` (wsm-spaces 01 and 02 merged), and against two pieces of
work that run before it (decision 15):

- **job `wsm-03`** (`units/wsm-spaces/03-skills-and-links/`): renames `PSM_CLI_VERSION` to
  `ANFS_SKILLS_CLI_VERSION` in `anfs.conf` (its commit `905d5ba` on the job's branch), read by the
  new `lib/anfs/skill.sh` for psm and wsm. This unit runs after it merges.
- **linker plan 02** (`units/linker/02-links-and-files/plan.md`): removes `PPM_GROUP_ID` (and
  install.sh's seed of it), and adds `link_dirs:`, so `~/.local/lib/anfs`, `lib/pcm`, `lib/pmm`,
  stack's `lib/tsm` and `lib/rpi-imager` become one link to the directory, and deletes ppm's and
  wsm's fallbacks to the core clone's `link.sh`. `lib/ppm` stays per file (core/ppm, anfs-dev/dev
  and stack/ruby all write there). The plans here are written as if it has merged.

Depends on: `wsm-spaces/03-skills-and-links` and `linker/02-links-and-files` merged.

## What is there today (core main + wsm-03)

- **`lib/anfs/paths.sh`** sets `ANFS_*` paths, `ANFS_CONF` and `ANFS_LOCAL_CONF`, and calls
  `anfs_load_conf` at source time: `anfs.local.conf` then `anfs.conf`, `NAME=value` lines read
  (not sourced), a quoted value unquoted, each exported only when unset. So environment > local >
  shipped, and every tool gets every tool's settings. `anfs_tool_paths <tool>` sets
  `<TOOL>_{CONFIG,DATA,STATE,CACHE}_HOME`. No test covers `anfs_load_conf`.
- **`anfs.conf`** (`packages/anfs/home/.config/anfs/anfs.conf`) is linked to
  `~/.config/anfs/anfs.conf`. A higher layer replaces it whole: `user/anfs` and `rws/anfs` each ship
  one (this Mac links core's; user's differs only in `ANFS_QUIET_SKIPPED_REPOS=true`).
- **install.sh** `install_configs` writes `anfs.local.conf` with `PPM_GROUP_ID=<os>` on a fresh
  machine. This Mac's `anfs.local.conf`: `PPM_GROUP_ID` and `PCM_MACHINE_MEMORY`.
- **Every entry script** starts `ANFS_LIB="$HOME/.local/lib/anfs"`, sources `bash.sh` (the bash
  floor, which may re-exec `"$0"` under bash 5), then `paths.sh`, then the rest from `$ANFS_LIB`.
  ppm spells `$HOME/.local/lib/anfs/...` out each time and loads `$HOME/.local/lib/ppm/*.sh`
  (its own and other packages' extensions: anfs-dev's `container.sh`, `user.sh`, `vm.sh`,
  `move.sh`; stack/ruby's `ruby.sh`). pcm and pmm find their own libraries with `pcm_lib_dir` /
  `pmm_lib_dir`: `$HOME/.local/lib/<tool>` when present, next to the script's real path otherwise.
  stack's `tsm` and `rpi-imager` do the same with their own lists. Tools on `lib/anfs`: core's
  anfs, ppm, pcm, pmm, psm, wsm; anfs-dev's `anfs-work`, `anfs-hooks`; stack's `opm`, `remotefs`,
  `ssh-key`, `tsm`, `pikvm`, `ventoy`, `ovm`, `gem-local`, `git-account` (and user's `pjm`,
  outside this unit). `rpi-imager`, `firstmate` and `bip39` do not source it.
- **pcm**: `PCM_ENV_HOME` (`~/.config/pcm/env`); `export_env_overrides` (`runtime.sh`) exports
  `<name>.env`'s lines below the environment, before varlock resolves `.env.schema`;
  `ingress_hosts` reads it; `pcm show` (`inspect.sh`) prints its path; the cloudflare-tunnel
  edge's `write_token` writes `TUNNEL_TOKEN` into `cloudflared.env` on `pcm up`. The local source
  is `$PCM_CONTAINERS_HOME/<name>/compose.yml` (`catalog_load ... compose.yml`, so the catalogue
  already needs the marker); `complete_service` lists every directory there, marker or not.
  **There is no `pcm expose` command** in main: `ingress.sh`'s comment names one (see
  "Conflicts").
- **`anfs work`** (anfs-dev): a job is `~/work/<job>/` (`$ANFS_WORK_HOME`) with a worktree per
  repo, named by the source's alias (`core/`, `stack/`, `anfs-dev/`). `anfs work run` and `env`
  set `ANFS_WORK_JOB`, `ANFS_WORK_NUMBER`, `PMM_SOCK_DIR`, `PMM_SSH_PORT_BASE`, `TMPDIR`,
  `ROUNDTRIP_SRC_ROOT` for the job's tests. Nothing reads `ANFS_WORK`.

## Every setting in `anfs.conf`, and where it goes

| Setting | Read by | Today | Goes to |
| --- | --- | --- | --- |
| `ANFS_UPDATE_CACHE_DURATION` | `lib/anfs/sources.sh` (every tool's source updates) | `86400`, active | `anfs.env` |
| `ANFS_QUIET_SKIPPED_REPOS` | `lib/anfs/sources.sh` | commented (`true` in user/anfs's) | `anfs.env` |
| `HOMEBREW_NO_AUTO_UPDATE` | brew, through the environment | `1`, active | `anfs.env` |
| `HOMEBREW_NO_ENV_HINTS` | brew | `1`, active | `anfs.env` |
| `HOMEBREW_UPDATE_CACHE_DURATION` | ppm `platform.sh` (also in install.sh's bootstrap libs) | `86400`, active | renamed `PPM_BREW_UPDATE_CACHE_DURATION`, in `ppm.env` (decision 8; no shim) |
| `ANFS_SKILLS_CLI_VERSION` (was `PSM_CLI_VERSION`, wsm-03) | `lib/anfs/skill.sh`, for psm and wsm | commented, `latest` | `anfs.env` (an `ANFS_` setting: two tools share it) |
| `PCM_COMPOSE` | pcm | commented | `pcm.env` |
| `PCM_HEALTH_TIMEOUT` | pcm `runtime.sh` | commented, `120` | `pcm.env` |
| `PCM_SHARED_NETWORK` | pcm `runtime.sh`, `ingress.sh`, `implode.sh` | commented, `dev-net` | `pcm.env` |
| `PCM_ATTACHED_SERVICES` | pcm `runtime.sh` | commented | `pcm.env` |
| `PCM_MACHINE_MEMORY` | pcm `runtime.sh` | commented (set in this Mac's `anfs.local.conf`) | `pcm.env`; this Mac's value to `pcm.local.env` by hand |
| `PCM_PROXY` | pcm `ingress.sh` | commented | `pcm.env` |
| `PCM_INGRESS_DOMAIN` | pcm | commented, `localhost` | `pcm.env` |
| `PMM_INSTALL` | pmm | commented, `up` | `pmm.env` |
| `PMM_SSH_PORT_BASE` | pmm `core.sh` (and set per job by `anfs work`) | commented, `2222` | `pmm.env` |
| `PMM_SSH_TIMEOUT` | pmm `machine.sh` | commented, `600` | `pmm.env` |
| `PMM_IMAGE_INSTALL_TIMEOUT` | pmm `build.sh` | commented, `3600` | `pmm.env` |
| `PMM_IMAGE_SSH_TIMEOUT` | pmm `build.sh`, `image-test.sh` | commented, `600` | `pmm.env` |
| `WSM_SPACES_HOME` | wsm (expands `~` itself) | commented, `~/spaces` | `wsm.env` |
| `PPM_GROUP_ID` (in `anfs.local.conf` only, seeded) | ppm `installer.sh`, anfs-dev `move.sh` | `<os>` | gone with linker 02, before this unit; a leftover line in this Mac's `anfs.local.conf` is dropped by hand |

Not in any file and staying so: the `*_HOME` paths (decision 1, "Decided here"), `ANFS_WORK`,
`ANFS_WORK_HOME` and `ANFS_WORK_TMP` (decision 11).

stack's opm settings are renamed in the same move (decision 10): `OP_SHARED_VAULT`,
`OP_SA_CREDENTIALS_VAULT`, `OP_PROVISION_TAG` become `OPM_SHARED_VAULT`,
`OPM_SA_CREDENTIALS_VAULT`, `OPM_PROVISION_TAG` (`OPM_TAG` stays), in `opm.env`.

## Decisions (taken with the person, 2026-10-21)

1. **Settings per tool.** A tool's package ships `~/.config/<tool>/<tool>.env` (every setting the
   tool reads, as a commented default with a line saying what it does; linked per file); a
   machine's overrides go in `~/.config/<tool>/<tool>.local.env`. anfs's own is
   `~/.config/anfs/anfs.env` + `anfs.local.env`: the `ANFS_*` settings and the bootstrap's
   `HOMEBREW_*` (brew's own two; see decision 8 for the third). `anfs.conf` and `anfs.local.conf` go. A tool not installed contributes nothing
   (this fixes the encapsulation: the `system` stack installs only some tools). Precedence:
   environment > local > shipped > built-in default. Format: `KEY=value` lines, parsed, never
   sourced. `anfs_load_conf` loads anfs's files and the running tool's (through
   `anfs_tool_paths <tool>`). Stack tools with settings follow the same rule. No migration code:
   the person moves `anfs.local.conf`'s lines by hand.
2. **A `config` command group** (`list`, `get NAME`, `set NAME=VALUE`, `unset NAME`, `list
   --export` for `eval`; FROM one of `env`, `local`, `shipped`, `default`; a table and `--json` in
   cli/01's shape). `config get|set` is the convention (git, npm, gh, pnpm); `env` means "print for
   eval" (`mise env`, `anfs work env`), so not `env`. *Revised after review (2026-10-21), replacing
   a single `anfs config` over every tool:* `lib/anfs/config.sh` supplies the group and every tool
   declares it through `cli.sh`, scoped to that tool: `pcm config set PCM_HEALTH_TIMEOUT=300`
   writes `~/.config/pcm/pcm.local.env`. The `anfs` seam gives `anfs container config …` for free.
   `anfs config` covers anfs's own settings (`ANFS_*`, `HOMEBREW_*`); `anfs config list --all`
   lists every installed tool's, with a TOOL column, through the library in one process. stack's
   tools on `cli.sh` (ovm, tsm, ventoy, remotefs, pikvm, opm) get it too. Names are full (the
   setting is the environment variable). A name the tool does not declare is refused; there is no
   owner lookup by prefix. Why: each tool owns its settings, a tool that is not installed has no
   command, and a tool's own command is where a user looks first. No tool has a `config` command
   today (checked).
3. **`ANFS_WORK=<job>`**: one setting to run a job's tools on live data. An installed tool, first
   thing (before sourcing anything else), checks `ANFS_WORK`; if the job has its own copy of the
   tool, it `exec`s it. Library directories derive from one root in `paths.sh` (no hard-coded
   `$HOME/.local/lib/...` in the tools), which is the job's checkout when `ANFS_WORK` is set; the
   installed `paths.sh` hands over to the job's `paths.sh` first. Tools not in the job run as
   installed. One stderr line per run: `anfs: running job <job>`. Honoured from the environment
   only, never from a settings file. `anfs config list` shows it. It replaces any `anfs work use` idea.
   The hand-over is tiny and identical in every tool (a shared preamble), and covers stack's tools
   on `lib/anfs`.
4. **pcm's per-container settings**: `PCM_ENV_HOME` and `~/.config/pcm/env/` go. A container's
   settings are `~/.config/pcm/containers/<name>/local.env`. A directory under `containers/` is a
   definition only when it has a `compose.yml`; one with only `local.env` is settings for a
   container defined in any source (the catalogue and completion skip it). Precedence:
   environment > `local.env` > the definition's `.env.schema` defaults (decision 9 adds pcm's state
   file below `local.env`). What a user's command writes for a container (`pcm expose`, when it
   exists) writes there. The person moves `env/ui.env` by hand.
5. **The XDG rule**, written into core `AGENTS.md`: CONFIG is what the user declares (by hand, or
   by an explicit `set`/`add` command, never as a side effect); STATE is what a tool records about
   what it did; DATA is content a tool manages; CACHE is rebuildable; RUNTIME is sockets and pids
   (`$XDG_RUNTIME_DIR` on Linux, `/tmp` on macOS). Plan 05 audits every write into a
   `*_CONFIG_HOME` across core and stack and fixes each that is a side effect.
6. **No backward compatibility and no migration code anywhere.** Each plan's Verification lists
   the by-hand steps "for the person after merge".
7. **Local definition directories stay in CONFIG**, for pcm and pmm alike:
   `~/.config/pcm/containers/<name>/` and `~/.config/pmm/machines/<name>/` (and pmm's
   `machines/images/`, `machines/networks/`) remain the unversioned, highest-priority `local`
   source. A definition is what the user declares; the precedent is podman Quadlet
   (`~/.config/containers/systemd`) and systemd user units. What a container keeps (volumes)
   stays DATA (`PCM_VOLUMES_HOME`). A container's settings live beside its definition
   (`local.env`: the compose convention of one tree per project). Plan 05's audit confirms nothing
   DATA-like lives under a `*_CONFIG_HOME`.

### After review (2026-10-21): the draft's open questions

8. **`HOMEBREW_UPDATE_CACHE_DURATION` becomes `PPM_BREW_UPDATE_CACHE_DURATION`**, in `ppm.env`
   (only ppm reads it; it is not a Homebrew variable). No shim.
9. **The cloudflare-tunnel token goes to STATE**: a file in pcm's state directory
   (`$PCM_STATE_HOME/containers/<name>/state.env`, mode 0600), read below `local.env`. It is
   fetched and can be fetched again, so it is STATE. Precedence: environment > `local.env` >
   `state.env` > `.env.schema` defaults. Plan 05 does it.
10. **opm's `OP_*` settings become `OPM_*`.** No shim.
11. **`ANFS_WORK_HOME` and `ANFS_WORK_TMP` are environment-only** (the hand-over reads
    `ANFS_WORK_HOME` before any file).
12. **user's `pjm` gets the preamble**: plan 03 includes the user repo for that one line.
13. **anfs-dev's git hooks** in `~/.config/git/ppm-hooks` (plan 05's finding 11): out of scope,
    in `BACKLOG.md`.
14. **stack/gam's `gam.cfg` and stack/ssh's `Include` line stay**, noted as future `setup:` steps
    (`units/setup.md`).
15. **Order**: `wsm-spaces/03` → `linker/02-links-and-files` → config 01 → 02, 03, 04 → 05 any
    time. linker 02 first so the library directories are directory links and plan 03 maps a
    directory, not each file (`lib/ppm`, shared by three packages, stays per file and keeps its
    case).

## Decided here (within the decisions above)

- **Where a tool's files are**: `${<TOOL>_CONFIG_HOME:-$XDG_CONFIG_HOME/<tool>}/<tool>.env` and
  `<tool>.local.env`. A test that points `PCM_CONFIG_HOME` at a temp dir gets that dir's files.
  The `*_HOME` variables locate the files, so they are environment-only: a `PCM_CONFIG_HOME=` line
  in `pcm.env` is ignored with one warning. `WSM_SPACES_HOME` is a setting (a place wsm links
  into), not one of these.
- **The loader**: `anfs_load_conf [tool]` in `paths.sh`. Without a tool (paths.sh calls it at
  source time) it loads anfs's two files; `anfs_tool_paths <tool>` ends by calling `anfs_load_conf
  <tool>`. Local first, then shipped, each name exported only when unset (today's rule, so the
  environment wins). A line naming `ANFS_WORK` is skipped with one warning. A package hook that
  needs its tool's setting calls `anfs_load_conf <tool>` itself (stack/tailscale's `TSM_AUTOJOIN`
  runs inside ppm, which loads only anfs's and ppm's).
- **"default"** in `config list` is the commented `#NAME=value` line of the shipped file: the
  documented built-in default. A shipped file therefore documents every setting its tool reads;
  a test holds each core tool to it (every `${PCM_X:-…}`/`: "${PCM_X:=…}"` that is a setting has a
  line in `pcm.env`). Internal switches (`*_IN_FNOX`, `*_BASH_REEXEC`, `*_LIB`, test hooks) are not
  settings and are not listed.
- **What a tool declares** (for `get`/`set`/`unset`): the names in its shipped file, active or
  commented. Anything else is refused (64). `HOMEBREW_NO_AUTO_UPDATE`, `HOMEBREW_NO_ENV_HINTS` and
  `ANFS_*` are anfs's.
- **Which tools declare the group**: those that ship a `<tool>.env` (core's anfs, ppm, pcm, pmm,
  wsm; stack's ovm, tsm, ventoy, remotefs, pikvm, opm). psm (its one setting is anfs's
  `ANFS_SKILLS_CLI_VERSION`), anfs-dev's `anfs-work`/`anfs-hooks` and stack's `ssh-key`,
  `gem-local`, `git-account` read no setting of their own and have no group; a tool that gains a
  setting gains its `.env` and the group together (a registry test holds the two together).
- **Which tools `anfs config list --all` shows**: anfs, then every directory under
  `$XDG_CONFIG_HOME` holding `<dir>/<dir>.env` (a linked shipped file is an installed tool); a
  `<TOOL>_CONFIG_HOME` set in the environment is honoured for that tool. No list of tools in
  anfs: a stack tool shows up when it is installed.
- **The hand-over**: a two-line preamble in every entry script, before `bash.sh`, that sources
  `lib/anfs/work.sh` only when `ANFS_WORK` is set (plan 03 has the text and the rules). Only the
  installed copy (run as `$HOME/.local/bin/<tool>`) ever hands over, so a job's copy, a test's copy
  or a checkout run by path never does. `anfs work run` and the test helpers unset `ANFS_WORK`, so
  a person's exported `ANFS_WORK` never reaches another job's tests.
- **`ANFS_WORK` names a job that does not exist**: the tool stops (exit 64, `anfs: no job <job> in
  ~/work`), rather than silently running the live code.

## Conflicts between the decisions and the code

- **"One root" for libraries (decision 3)** holds for the installed tree (`$HOME/.local/lib`), not
  for a job: there the libraries are in each package's `home/.local/lib/<name>/`, across repos
  (`lib/anfs` in core's `anfs` package, `lib/tsm` in stack's `tailscale`), and `lib/ppm` is filled
  by three packages. Plan 03 keeps one root in `paths.sh` (`ANFS_LIB_ROOT`) and maps each library
  directory to the job through its installed link: with linker 02 merged (decision 15) a library
  directory is one link into `sources/<alias>/<path>`, and the job's copy is `<alias>/<path>` in
  the job when the job has that repo. `lib/ppm` stays per file: the job's copy of each installed
  file, plus the job's new files in `core/packages/ppm`.
- **`pcm expose` (decision 4) does not exist** in main; only a comment in `ingress.sh` names it.
  Plan 04 points every writer that exists (the cloudflare-tunnel edge) at `local.env` and fixes
  the comment; if `pcm expose` lands first (another unit), it writes `local.env` too.
- **The edge's `TUNNEL_TOKEN` is written into CONFIG on `pcm up`** (a side effect, decision 5),
  into the very file decision 4 makes the user's. Plan 05 moves it to STATE (decision 9); if plan
  04 runs first, it points the edge at `local.env` meanwhile and plan 05 moves it on.
- **`bash.sh` runs before the preamble's alternative**: the preamble must come first, under
  macOS's bash 3.2, so `work.sh` joins the bootstrap set (3.2-safe, parsed under 3.2 by
  `bash.bats`) beside `bash.sh`.
- **A layer's `anfs.conf`** (user/anfs, rws/anfs) replaces core's file whole. Under decision 1 a
  layer may ship `anfs.env` the same way, but a machine's overrides belong in `anfs.local.env`.
  The person decides per layer (by-hand step, plan 01).
- **stack's `tsm`** sets `TSM_CONFIG_HOME` to tailscale's directory (`~/.config/tailscale`), not
  its own. Plan 01 renames that variable `TSM_TAILSCALE_HOME` (no shim), so `TSM_CONFIG_HOME` is
  tsm's own and holds `tsm.env`; tailscale's config stays where it is. Its install hook names a file
  that never existed (`~/.config/ppm/ppm.local.conf`); plan 01 corrects it to `tsm.local.env`.
- **stack's `opm`** reads `OP_SHARED_VAULT`, `OP_SA_CREDENTIALS_VAULT`, `OP_PROVISION_TAG`,
  `OPM_TAG`: names not all prefixed by the tool. Renamed (decision 10).

## Plans

| # | Plan | What | Job repos |
| --- | --- | --- | --- |
| 01 | `01-tool-env` | `anfs_load_conf [tool]`; every tool's `<tool>.env` (core's anfs, ppm, pcm, pmm, wsm; stack's remotefs, ventoy, pikvm, ovm, tsm, opm); `PPM_BREW_UPDATE_CACHE_DURATION`, opm's `OPM_*`; `anfs.conf` and `anfs.local.conf` gone; install.sh seeds nothing; docs | core, stack, anfs-dev |
| 02 | `02-config-command` | `lib/anfs/config.sh`: the `config` group (`list\|get\|set\|unset`, `list --export`, `--json`) in every tool on `cli.sh`, core's and stack's; `anfs config list --all`; `ANFS_WORK` shown | core, stack |
| 03 | `03-work-switch` | `ANFS_WORK=<job>`: `lib/anfs/work.sh`, the preamble in every tool, `ANFS_LIB_ROOT` and the per-directory job mapping in `paths.sh` (`lib/ppm` per file), `anfs work run` unsets it | core, stack, anfs-dev, user (pjm's line) |
| 04 | `04-pcm-local-env` | `containers/<name>/local.env`; `PCM_ENV_HOME` gone; catalogue and completion skip settings-only dirs; `pcm show`, docs, the `pcm-containers` skill | core, stack |
| 05 | `05-xdg-rule` | The rule in core `AGENTS.md`; the audit of every write into a config directory and its fixes; the tunnel token to STATE; pmm's sockets under `$XDG_RUNTIME_DIR` on Linux | core, stack |

The order (decision 15): `wsm-spaces/03` → `linker/02-links-and-files` → 01 → 02, 03, 04 side by
side → 05 any time. 02 reads 01's files and edits every tool's command declarations; 03 edits every
entry script's header and `paths.sh` (02 and 03 both touch the tools' scripts, in different
places: whichever merges second rebases); 04 uses 01's loader and edits pcm. 03 needs linker 02's
directory links. 05 and 04 both touch pcm's env reading: whichever merges second rebases (05
adds `state.env` below whatever file 04 has made the user's).

## Open questions

None left: the draft's seven were answered on 2026-10-21 (decisions 8–14).

## Completion Criteria

- `grep -rn 'anfs\.conf\|anfs\.local\.conf\|ANFS_LOCAL_CONF\|PCM_ENV_HOME\|pcm/env'` in core,
  stack and anfs-dev finds only the units' history (nothing in code, READMEs, AGENTS.md, skills).
- With only `core/anfs` and `core/ppm` installed, no `PCM_*` or `PMM_*` setting is in a tool's
  environment; with `core/pcm`, `PCM_HEALTH_TIMEOUT=300 pcm …` beats `pcm.local.env`, which beats
  `pcm.env`.
- `pcm config list` on a temp home shows pcm's settings with FROM; `pcm config set
  PCM_HEALTH_TIMEOUT=300` (or `anfs container config set …`) writes `~/.config/pcm/pcm.local.env`;
  `get` reads it back; `unset` removes the line; `eval "$(pcm config list --export)"` sets them;
  `pcm config set PMM_INSTALL=x` is refused; `anfs config list --all` shows every installed tool's.
- `ANFS_WORK=<job> pcm list` runs the job's pcm and the job's `lib/anfs` against the live data,
  says `anfs: running job <job>` once on stderr, and a tool the job lacks runs as installed;
  `ANFS_WORK` in `anfs.local.env` has no effect.
- The tunnel token is in `$PCM_STATE_HOME`, mode 0600, and nothing under `~/.config/pcm` is
  written by `pcm up`.
- `pcm show <name>` names `~/.config/pcm/containers/<name>/local.env`; a directory there with only
  `local.env` is not in `pcm list` or completion.
- Core `AGENTS.md` has the XDG rule; plan 05's log lists every finding and what was done with it.
