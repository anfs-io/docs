---
plan: config/01-tool-env
job: config-01
date: 2026-10-22
---

# Log: config/01-tool-env

## The plan against current main

The plan was written before linker/02 merged. Checked against main (core `ea62700`, stack
`2408e71`):

- `PPM_GROUP_ID` is gone (grep in core: only `pcm/legacy/README.md`, AGENTS.md saying it is gone,
  ppm's test that it changes nothing). install.sh's seed after linker/02 wrote `anfs.local.conf`
  with a header line only; that seed is now removed entirely.
- `lib/anfs` is one directory link, so `paths.sh`'s new loader is live the moment core is pulled.
  The shipped `.config/<tool>/<tool>.env` files are linked **per file** (no `links:` entry), so
  each needs a `ppm install` of its package (by-hand steps below).
- `platform.sh` moved to `lib/anfs`, but `update_brew_if_needed` stayed in ppm's
  `lib/ppm/platform.sh` (still in `PPM_BOOTSTRAP_LIBS`): that is where the rename happened.
  install.sh's bootstrap gets the default (86400), as the plan says.
- Nothing else in the plan depended on linker/02's details.

## What was built

- **The loader** (`lib/anfs/paths.sh`, "Settings"; the API plans 02–04 build on):
  - `anfs_conf_files <tool>`: `<dir>/<tool>.local.env` then `<dir>/<tool>.env`, `<dir>` =
    `${<TOOL>_CONFIG_HOME:-$XDG_CONFIG_HOME/<tool>}` (anfs: `$ANFS_CONFIG_HOME`; a `-` in a tool
    name becomes `_` in the variable).
  - `anfs_conf_read <file>`: `NAME<TAB>value` per setting line, file order, nothing exported.
    Blanks and comments skipped, a leading `export ` accepted, one pair of matching quotes
    stripped, nothing expanded, a malformed line skipped.
  - `anfs_conf_env_only <name>`: 0 for `*_CONFIG_HOME`, `*_DATA_HOME`, `*_STATE_HOME`,
    `*_CACHE_HOME`, `ANFS_WORK`, `ANFS_WORK_HOME`, `ANFS_WORK_TMP`.
  - `anfs_load_conf [tool]` (no tool = anfs): local then shipped, each name exported only when
    unset; an environment-only name is skipped with one warning per line (`anfs: <file>: <NAME>
    is read from the environment only; the line is ignored`). Parsing fills two arrays: no fork
    per file.
  - `paths.sh` loads anfs's at source time (moved to the end of the file, after `anfs_display`);
    `anfs_tool_paths <tool>` ends with `anfs_load_conf <tool>` (and uses `${1^^}`, not `tr`).
  - `ANFS_CONF`, `ANFS_LOCAL_CONF` gone.
- **Shipped files** (one header, the same words in each; then a comment and a line per setting):
  - core: `anfs/anfs.env` (`ANFS_UPDATE_CACHE_DURATION=86400`, `HOMEBREW_NO_AUTO_UPDATE=1`,
    `HOMEBREW_NO_ENV_HINTS=1` active; `ANFS_QUIET_SKIPPED_REPOS`, `ANFS_SKILLS_CLI_VERSION`,
    `ANFS_RPC_PORT`, `ANFS_RPC_TIMEOUT`, `ANFS_LOG_MAX` commented), `ppm/ppm.env`
    (`PPM_BREW_UPDATE_CACHE_DURATION=86400` active, `PPM_DEFAULT_REPO`), `pcm/pcm.env` (the
    table's seven plus `PCM_MACHINE_HEADROOM`, `PCM_TUNNEL_NAME`), `pmm/pmm.env` (the five plus
    `PMM_IMAGE_VERBOSE`, `PMM_DISPLAY`, `PMM_LIBVIRT_URI`), `wsm/wsm.env` (`WSM_SPACES_HOME`). No
    `psm.env`. `anfs/anfs.conf` deleted.
  - stack: `remotefs.env`, `ventoy.env`, `pikvm.env`, `tsm.env` (stack/tailscale), `opm.env`
    (stack/op), `ovm.env` (stack/obsidian).
  - (Before merge: `OPM_PROVISION_TAG` folded into `OPM_TAG`; the loader expands a leading `~/`.
    See "Fixes before merge".)
- **ppm**: `HOMEBREW_UPDATE_CACHE_DURATION` → `PPM_BREW_UPDATE_CACHE_DURATION` (no shim); header
  comments.
- **install.sh**: `install_configs` seeds `user.list` only; the header's steps and file list,
  `stow_base`'s and `install_repo`'s comments name `anfs.env`.
- **stack**:
  - pikvm, tsm, opm call `anfs_tool_paths <tool>`.
  - tsm: `TSM_CONFIG_HOME` (tailscale's directory) → `TSM_TAILSCALE_HOME`, no shim; `TSM_CONFIG`
    defaults under it; `TSM_CACHE` defaults to `$TSM_CACHE_HOME` (same path). Help lists the
    settings and the paths separately.
  - tailscale's `post_install` calls `anfs_load_conf tsm` before reading `TSM_AUTOJOIN`; its
    message, comment and README name `~/.config/tsm/tsm.local.env`. The old comment ("conf vars
    are NOT exported") is gone: they are exported now.
  - opm: `OP_SHARED_VAULT`, `OP_SA_CREDENTIALS_VAULT`, `OP_PROVISION_TAG` → `OPM_*` in code, help,
    README, tests (no shim; `OPM_TAG` stays). opm's hook never read them.
  - Help epilogues of remotefs, ventoy, pikvm say "Settings (~/.config/<tool>/<tool>.env ...)";
    READMEs of remotefs, ventoy, obsidian, tailscale, op one line each (pikvm has no README);
    stack AGENTS.md: a tool with settings calls `anfs_tool_paths` and ships its file.
- **Other readers of the old names**: `pcm.zsh`'s `cconf` defaulted to `../../anfs/anfs.local.conf`;
  now `../pcm.local.env`. `ingress.sh`'s message says `PCM_PROXY in pcm.local.env`. wsm's
  `_spaces_home` comment, wsm's `install.sh`, `sources.sh`, `skill.sh` comments.
- **Docs**: core AGENTS.md "Configuration: `<tool>.env`" rewritten (files, precedence, who loads
  what, a hook's `anfs_load_conf <tool>`, format, environment-only names, naming, the API, a
  layer's `anfs.env`, `<tool> config` coming), Repository Layout, `wsm: spaces`' one root, Package
  Structure, Key Files, Your Customization Repo, Claiming (`PPM_DEFAULT_REPO`), Lib Structure
  (`paths.sh`), Testing; README, anfs/ppm/wsm READMEs, psm AGENTS.md, the `pcm-containers` skill;
  anfs-dev's `packages/dev/README.md` (`ANFS_WORK_HOME` environment only).

`grep -rn 'anfs\.conf\|anfs\.local\.conf\|ANFS_LOCAL_CONF\|ANFS_CONF\b'` in core, stack and
anfs-dev finds only the tests that prove they are not read. `HOMEBREW_UPDATE_CACHE_DURATION` only
in ppm's test that it is not read. `OP_SHARED_VAULT|OP_SA_CREDENTIALS_VAULT|OP_PROVISION_TAG` only
in opm's test that they are not read; `TSM_CONFIG_HOME` nowhere in tailscale but the test's unset.

## Commits (branch config-01, not pushed)

core:
- `e060e77` settings per tool: the loader, the shipped files, `anfs.conf` gone, ppm's rename,
  install.sh, `conf.bats`, `settings.bats`, ppm `brew.bats`
- `d063939` docs (AGENTS.md, READMEs, psm AGENTS.md, pcm-containers skill)

stack:
- `d2a12b8` remotefs, ventoy, pikvm, tsm, opm, ovm: `<tool>.env`; `OPM_*`; `TSM_TAILSCALE_HOME`;
  the hook; tests; READMEs, AGENTS.md

anfs-dev:
- `683331f` dev README: `ANFS_WORK_HOME` from the environment only

Versions: the pre-commit hook bumped psm 0.1.7 (core), pikvm 0.1.2, remotefs 0.1.1, ventoy 0.1.3
(stack), dev 0.1.15 (anfs-dev). core's anfs, ppm, pcm, pmm, wsm and stack's obsidian, op,
tailscale already carry an unpushed bump (linker/02's, merged into local main, not pushed), so the
hook left them (one bump per unpushed series). `WSM_VERSION` and `OVM_VERSION` unchanged (no
command changed; `OVM_VERSION` still equals obsidian's 0.2.1).

## Decisions made during the work

- **More settings than the plan's lists**, found by the settings test's scan (each is a
  `${X:-…}`/`:=` a person would set): `ANFS_RPC_PORT`, `ANFS_RPC_TIMEOUT`, `ANFS_LOG_MAX`,
  `PPM_DEFAULT_REPO`, `PCM_MACHINE_HEADROOM`, `PCM_TUNNEL_NAME`, `PMM_IMAGE_VERBOSE`,
  `PMM_DISPLAY`, `PMM_LIBVIRT_URI`, `REMOTEFS_MOUNT_HOME` (a place it mounts into, like
  `WSM_SPACES_HOME`).
- **`VENTOY_DISK` is a setting** (the plan suspected per run): ventoy's help calls it "Default
  disk, e.g. /dev/disk/by-id/usb-... (stable across replugs)" and `ventoy path` prints it, so it is
  a machine's value. In `ventoy.env`, commented and empty.
- **Not settings** (`settings.bats`'s allow-list, each with why): anfs's switches and per-caller
  knobs (`ANFS_DEBUG`, `_JSON`, `_TABLE`, `_TRACE`, `_COLOR`, `_LOG`, `_MF_*`, `_NAME_NEST`,
  `_RESOLVE_*`, `_RPC`, `_RPC_OS`, `_BASH_*`, `_TEST_TERMINAL`); ppm's `PPM_CURRENT_PACKAGE`,
  `PPM_DEBUG`, `PPM_SKIP_FILE`; pcm's `PCM_PODMAN_SOCKET` (computed), `PCM_SERVICE` (set for a
  hook), `PCM_TOOLKIT_OS` (test hook); pmm's host facts and the files pmm-network sets up
  (`PMM_BRIDGE_CONF`, `_BRIDGE_HELPER`, `_SYS_NET`, `_NET_BACKEND`, `_NET_CHECK`, `_KVM_DEV`,
  `_HOST_OS`, `_HOST_ARCH`, `_VIRSH`), the probed firmware (`PMM_FW_CODE`/`_VARS`), tart's
  `PMM_MACOS_LIMIT`, the per-machine `PMM_SSH_HOST`, the path `PMM_SSH_CONFIG`; and every path
  that locates a tree (`PCM_CONTAINERS_HOME`, `PMM_MACHINES_HOME`, ...; `REMOTEFS_CONFIG`,
  `VENTOY_HOME`, `TSM_TAILSCALE_HOME`, `TSM_CONFIG`, `TSM_CACHE`); `TAILSCALE_BIN`.
- **A commented line is the code's default, not an example** (the old `anfs.conf` mixed them):
  `#ANFS_QUIET_SKIPPED_REPOS=false` (was `true`), `#PCM_MACHINE_MEMORY=` (was `4096`),
  `#PCM_PROXY=` (was `traefik`); the example goes in the comment. Plan 02's FROM `default` reads
  these lines, so they must be true.
- **Environment-only is a pattern**, not a list of what paths.sh sets: any
  `*_{CONFIG,DATA,STATE,CACHE}_HOME` (an `anfs.env` line `PCM_CONFIG_HOME=` would otherwise move
  pcm's files, since anfs's are loaded first), plus `ANFS_WORK_HOME` and `ANFS_WORK_TMP`
  (decision 11; plan 03 can rely on the loader never setting them).
- **Paths in a file** may start with `~/` (the loader's one expansion, added before merge, below);
  `VENTOY_ISO_DIR` is shipped as `#VENTOY_ISO_DIR=` with its XDG-derived default in the comment.
- **The settings test is `packages/anfs/tests/settings.bats`** (the plan allowed it), not
  `registry.bats`, which is `cli.sh`'s.
- **tsm.bats and opm.bats now run under a temp HOME with `ANFS_TEST_LIB`** like stack's other
  suites: they used the live `~/.local/lib/anfs`, so they could not test the new loader (and
  opm's read the real `~/.config/anfs`).
- `PMM_SSH_PORT_BASE` stays a setting in `pmm.env`; `anfs work` sets it per job in the
  environment, which wins.

## Tests (all through `anfs work run config-01 …`, `</dev/null`)

- While working: `conf.bats` 13 (new: the parser on quoted, unquoted, `export`, blank, comment and
  malformed lines; a missing file; `anfs_conf_files`, a moved `PCM_CONFIG_HOME`; env-only names;
  local over shipped, exported; the environment over both, empty included; `anfs.conf` not read;
  a tool with no directory; a second call; `ANFS_WORK`/`*_HOME` lines warned once and not
  exported; the encapsulation under `env -i`: only anfs's file, no `PCM_*`/`PMM_*`),
  `settings.bats` 8 (new: anfs, ppm, pcm, pmm, wsm each document what they read and nothing
  else; psm none; the rule catches a missing and an extra line; names prefixed, no `anfs.conf`
  shipped), ppm `brew.bats` 3 (new).
- **Once, at the end**: `packages/anfs/tests/all`: **526 passed, 0 failed**, 37 files (pmm and pcm
  off), 56 s.
- stack, `ANFS_TEST_LIB` = the job's core lib: remotefs 16/16, ventoy 18/18, pikvm 18/18 (new:
  `pikvm.local.env` over `pikvm.env`, the environment over both), obsidian 59/59, tailscale 28/29
  (new: the config under `TSM_TAILSCALE_HOME`, `TSM_AUTHKEY_MODE` from `tsm.local.env`, the hook
  joins only with `TSM_AUTOJOIN` in `tsm.local.env`), op 34/38 (new: `opm.local.env` sets the
  vaults, the environment wins, `OP_*` not read; `OPM_PROVISION_TAG`). **The 5 failures are the
  known ones on main**: `tsm.bats` "an option the command does not take is a usage error" and
  `ssh-hook.bats` 1, 7, 10, 11 (numbered 25, 31, 34, 35 in linker/02's run). Before any stack
  change, the same 5 failed with this job's core lib.
- By hand, a temp `HOME` under the job's `TMPDIR` with the job's core anfs, pcm, pmm linked into
  it, `env -i`: `pcm help` and `pmm help` run; `anfs_tool_paths pcm` leaves
  `PCM_HEALTH_TIMEOUT` unset with the shipped `pcm.env` (no `PCM_*` in the environment at all),
  `120` with the line active, `300` from `pcm.local.env`, `5` from the environment;
  `ANFS_WORK=x` in `anfs.local.env` prints the one warning and `pcm --version` runs.

## Not run

- pmm's and pcm's suites are off (ROADMAP, 2026-10-19) and were not run. pcm changes: `pcm.env`,
  a message in `ingress.sh`, `pcm.zsh`'s `cconf`; pmm: `pmm.env` only.
- anfs-dev's suites: only its README changed. anfs-ui: unchanged.
- Deferred to the end of the roadmap (boot a box or a VM): `roundtrip debian`, `roundtrip-macos`
  (a fresh install.sh: no `anfs.local.conf` written, `anfs.env` and `ppm.env` linked).
- A real install.sh run; nothing was installed on this host. `~/.config/anfs/*` was only read.

## Open questions

None left. Answered before merge (2026-10-22):

1. **`OPM_PROVISION_TAG` folded into `OPM_TAG`** (no shim): fixed, below.
2. **The loader expands a leading `~/`, or a value that is exactly `~`, to `$HOME`**, nothing else
   (no `$VAR`, no `~user`, no globbing, no `~` past the start): fixed, below.
3. The non-settings allow-list (test hooks, internal paths, pmm's paths) stays out of the `.env`
   files: agreed.
4. user/anfs's `anfs.conf` is dropped, not ported; `ANFS_QUIET_SKIPPED_REPOS=true` goes into this
   machine's `anfs.local.env` (step 4 below): agreed.

## Fixes before merge

- **The `~` expansion** (core `533935e`): `anfs_conf_expand <value>` (into `REPLY`, no fork),
  applied by `anfs_load_conf` to every value it exports, quoted or not. `anfs_conf_read` still
  returns values as written, so plan 02's `config` can show and rewrite a line as the person typed
  it, and expand with `anfs_conf_expand` where it needs the effective value. Documented in
  `paths.sh`'s Settings header and AGENTS.md "Configuration"; every shipped `.env` header says it.
  `REMOTEFS_MOUNT_HOME`'s shipped default is now `#REMOTEFS_MOUNT_HOME=~/mnt` (true now);
  `VENTOY_ISO_DIR` stays `#VENTOY_ISO_DIR=` (its default follows `XDG_CACHE_HOME`). No tool's own
  expansion was removed: wsm's `_spaces_home` stays (it also takes `$HOME`, and the environment
  can still carry a literal `~`); nothing else expanded `~` in a setting.
- **opm** (stack `2f989ee`): `: "${OPM_TAG:=op-provision}"`; `OPM_PROVISION_TAG` gone from code,
  help, `opm.env`, tests (README only ever named `OPM_TAG`).
- Tests: `conf.bats` 15 (new: `~/mnt`, a quoted `"~/x"`, a lone `~` and `'~'` expand; `~user`,
  `a/~/b`, `$HOME/x`, `~x` and a glob stay literal; `anfs_conf_read` keeps `~`;
  `anfs_conf_expand`), `settings.bats` 8, stack `opm.bats` (the tag test now: `OPM_TAG` from
  `opm.local.env`, `OP_PROVISION_TAG` and `OPM_PROVISION_TAG` not read), `remotefs.bats` 17 (new:
  `REMOTEFS_MOUNT_HOME=~/remote` in `remotefs.local.env` gives `$HOME/remote/<label>`).
  Stack again with the job's core lib: remotefs 17/17, ventoy 18/18, pikvm 18/18, obsidian 59/59,
  tailscale 28/29, op 34/38 (the same 5 known failures). **`packages/anfs/tests/all` once more:
  528 passed, 0 failed**, 37 files, 55 s.

## By hand, for the person after merge (in this order)

Between the pull and step 2 the tools read only what is linked: `anfs.env` is not yet, so
`HOMEBREW_NO_AUTO_UPDATE` and `HOMEBREW_NO_ENV_HINTS` are unset for that one install (hence the
prefix in step 2).

1. `anfs src update` (pulls core, stack, anfs-dev; `lib/anfs` is one link, so the new loader is
   live at once).
2. Link each tool's settings file (and drop core/anfs's `anfs.conf` link, which its tracker lists):

   ```bash
   HOMEBREW_NO_AUTO_UPDATE=1 ppm install core/anfs core/ppm core/pcm core/pmm core/wsm
   ls -l ~/.config/{anfs/anfs,ppm/ppm,pcm/pcm,pmm/pmm,wsm/wsm}.env
   ```

   (Leave `core/pmm` out if pmm is not installed here.)
3. Move this Mac's `~/.config/anfs/anfs.local.conf` (it holds, read 2026-10-22, a blank line and
   the two lines below; `PPM_GROUP_ID` is already gone):

   ```bash
   cat ~/.config/anfs/anfs.local.conf     # expect only the two lines below (and a blank one)
   mkdir -p ~/.config/pcm
   printf '%s\n' '# Podman machine memory (MiB); pcm up resizes the machine to it' 'PCM_MACHINE_MEMORY=4096' >> ~/.config/pcm/pcm.local.env
   rm ~/.config/anfs/anfs.local.conf
   [ -L ~/.config/anfs/anfs.conf ] && [ ! -e ~/.config/anfs/anfs.conf ] && rm ~/.config/anfs/anfs.conf
   ls -la ~/.config/anfs                    # system.list, user.list, anfs.env (a link)
   ```

4. **Your layers' `anfs.conf`**: user/anfs and rws/anfs each ship
   `packages/anfs/home/.config/anfs/anfs.conf` (never linked here: core's was). In each repo,
   delete that file and commit. user's one active difference from core's goes to this machine:

   ```bash
   echo 'ANFS_QUIET_SKIPPED_REPOS=true' >> ~/.config/anfs/anfs.local.env
   grep -v '^#' ~/.local/share/anfs/sources/rws/packages/anfs/home/.config/anfs/anfs.conf | grep .   # rws's active lines: put any you want in a *.local.env, by tool
   ```

   Then, once pulled: `ppm install user/anfs` (and `rws/anfs` if installed) so their trackers drop
   the file.
5. stack's tools with a settings file, each that is installed:

   ```bash
   for p in remotefs ventoy pikvm obsidian tailscale op; do
     [ -f ~/.local/state/ppm/installed/stack/$p.yml ] && ppm install stack/$p
   done
   ```

6. Renamed, no shim: if you set `OP_SHARED_VAULT`, `OP_SA_CREDENTIALS_VAULT` or `OP_PROVISION_TAG`
   (opm; the tag is now `OPM_TAG`) or `TSM_CONFIG_HOME` (tsm, meaning tailscale's directory) anywhere (shell rc, fnox,
   `.envrc`), rename them `OPM_SHARED_VAULT`, `OPM_SA_CREDENTIALS_VAULT`, `OPM_TAG`,
   `TSM_TAILSCALE_HOME`; `HOMEBREW_UPDATE_CACHE_DURATION` is `PPM_BREW_UPDATE_CACHE_DURATION`:

   ```bash
   grep -rnE 'OP_(SHARED_VAULT|SA_CREDENTIALS_VAULT|PROVISION_TAG)|TSM_CONFIG_HOME|HOMEBREW_UPDATE_CACHE_DURATION' ~/.zshrc ~/.zshenv ~/.bashrc ~/.config/sh ~/.config/zsh ~/.config/fnox 2>/dev/null
   ```

7. Check (bash 5: `paths.sh` is above the floor): `env -i HOME=$HOME /opt/homebrew/bin/bash -c 'source ~/.local/lib/anfs/paths.sh; anfs_tool_paths pcm; echo $PCM_MACHINE_MEMORY $HOMEBREW_NO_AUTO_UPDATE'` prints `4096 1`.

Not this plan's: `~/.config/pcm/env/ui.env` moves in plan 04.
