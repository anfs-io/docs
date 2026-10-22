---
---

# Plan 05 — The XDG rule, and every write into a config directory held to it

Written 2026-10-21 against core main `9e8cb73` plus job `wsm-03` and linker plan 02. Independent
of plans 01–04; it can run any time (unit decision 15). Where it meets 01 (install.sh's seed) or 04 (pcm's env files) it says which
plan owns the change.

## Context — read these files first

The job is `anfs work new <job> core stack`. Paths are core's unless they name another repo.

- `units/config.md` — decisions 5, 6, 7, 9 (the tunnel token to STATE), 13, 14
- `packages/pcm/home/.local/lib/pcm/runtime.sh` — `export_env_overrides` (where the state file is
  read); `edges/cloudflare-tunnel` `write_token`; `tests/edges.bats`; stack
  `containers/cloudflared/{compose.yml,.env.schema,README.md}` (they say where the token is)
- `AGENTS.md` — "Repository Layout" (the `<dir>/<tool>` paragraph), "Configuration", "pcm:
  containers", "pmm: machines"; `packages/anfs/home/.local/lib/anfs/paths.sh` (the header's layout)
- Each file of the findings table below, at the lines given
- `packages/pmm/home/.local/lib/pmm/core.sh` — `sock_dir_path`, `sock_dir` (lines ~50–70);
  `interfaces.sh` — `ssh_config_write` (~189–213); `implode.sh` (~37, 47)
- `packages/anfs/tests/isolation.bats` (the live socket directory pattern), pmm's tests that set
  `PMM_SOCK_DIR`; anfs-dev `anfs-work` `_job_env` (`PMM_SOCK_DIR` per job)
- stack `packages/ssh/install.sh` (the `Include config.d/*.conf` line), `packages/ruby/install.sh`
  and `home/.local/lib/ppm/ruby.sh` (`install_gem`), ruby's mise `conf.d`

## Overview

The rule is written where every agent reads it, and the toolkit keeps it: a tool writes into a
config directory only what the user declared, by hand or by an explicit `set`/`add` command.
What a tool records is STATE, what it manages is DATA, what it can rebuild is CACHE, and sockets
are RUNTIME. Local definitions (pcm's `containers/`, pmm's `machines/`) are CONFIG (decision 7).

## The audit (2026-10-21, against main; re-checked in this plan)

Every write into `$XDG_CONFIG_HOME`, a `*_CONFIG_HOME`, `~/.ssh` or another tool's config, by core
and stack tools, their install hooks and install.sh. **OK** writes (explicit declarations) are
listed so the next audit can diff against them.

| # | Where | Trigger | What | Verdict | Here |
| --- | --- | --- | --- | --- | --- |
| 1 | `install.sh:358` `install_configs` | install.sh | `anfs.local.conf` with `PPM_GROUP_ID` | side effect (a detected fact) | plan 01 removes it; recorded |
| 2 | `install.sh:353` `install_configs` | install.sh | `user.list` holding one comment | side effect (an empty template) | drop the seed if every reader tolerates a missing `user.list` (`sources.sh` creates it on `src add`); else keep and say why |
| 3 | `pcm/install.sh:22` | `ppm install core/pcm` | empty `~/.config/pcm/containers/` | side effect | drop the `mkdir`; nothing needs it (catalogue and completion tolerate it missing) |
| 4 | `pmm/install.sh:18` | `ppm install core/pmm` | empty `~/.config/pmm/machines/{images,networks}` | side effect | drop; `pmm image new` makes what it writes |
| 5 | `pcm/…/edges/cloudflare-tunnel:100–108` | `pcm up` of a service with a tunnel host | `TUNNEL_TOKEN` (fetched from the API) into the user's env file (`local.env` after plan 04) | side effect (a secret pcm fetched and can fetch again: STATE) | `write_token` writes `$PCM_STATE_HOME/containers/<name>/state.env` (mode 0600, other lines kept); `export_env_overrides` reads it below `local.env` (decision 9) |
| 6 | `pmm/…/interfaces.sh:189–213` `ssh_config_write` | every `pmm up`/`down` | `~/.ssh/config.d/05-pmm.conf`, a `Host` per running machine | side effect (generated from run state) | pmm writes `$PMM_STATE_HOME/ssh_config`; core/pmm ships `home/.ssh/config.d/05-pmm.conf` holding one `Include ~/.local/state/pmm/ssh_config` line (static, linked like any file); implode removes the state file |
| 7 | stack `ruby/install.sh:6`, `ruby.sh:4–5` `install_gem` | `ppm install stack/ruby` and packages that add gems | appends to `~/.config/ruby/default-gems` | side effect (a list packages contribute: DATA) | ruby's mise `conf.d` points `MISE_RUBY_DEFAULT_PACKAGES_FILE` at `$XDG_DATA_HOME/ruby/default-gems`; `install_gem` appends there |
| 8 | stack `ssh/install.sh:10` | `ppm install stack/ssh` | appends `Include config.d/*.conf` to `~/.ssh/config` | edit of the user's file at a location ssh dictates | kept (decision 14), documented as the one accepted edit (idempotent; it is what makes `config.d/` work); a future `setup:` step (`units/setup.md`) |
| 9 | stack `gam/install.sh:17,20` | `ppm install stack/gam` | `~/.config/gam7/gam.cfg` through `gam config drive_dir … verify` | side effect (a tool-chosen default, in the location GAM fixes) | kept (decision 14); a future `setup:` step with `units/setup` (not built here) |
| 10 | stack `firstmate:47–50` `trust_root` | each `firstmate` launch | `hasTrustDialogAccepted` in Claude Code's `.claude.json` | another tool's file (mostly its state) | kept, deliberate; documented in firstmate's README |
| 11 | anfs-dev `anfs-hooks:149,157`, `anfs-work:238` | `ppm install anfs-dev/dev`, `anfs hooks install`, `anfs work new` | `core.hooksPath` in each repo's `.git/config`; hook code shipped under `~/.config/git/ppm-hooks` | the hooksPath on install is a side effect; code under CONFIG | out of scope (decision 13): in `BACKLOG.md` (move the hooks to `~/.local/lib/anfs-dev/hooks`; set hooksPath only from `anfs hooks install`) |
| 12 | `anfs/…/listener.sh:135–158` | `ppm install core/anfs` | launchd plist (macOS), systemd user units (Linux) | platform (the OS dictates) | kept |
| 13 | `pcm/…/runtime.sh:319–326` | first `pcm up` on macOS | podman's own machine config | platform (podman's) | kept |
| 14 | `sources.sh` `src add/remove/ssh`; `onboard.sh` repo steps; `iso.sh` `iso add/remove --entry`; `ppm file claim/add/reset/protect`; `psm repo add/remove`; `pmm image new`; stack `remotefs add/remove/edit`, `gem-local src add/remove`, `git account add/remove`, git and op onboard steps, `opm create/link`; `ssh` onboard's `authorized_keys` | explicit commands | the user's declarations | OK | none |

Not writers into config: wsm, ovm, tsm, rpi-imager, pjm (user), ppm's backups (STATE), pcm's
overrides (CACHE), pmm's machines, networks and builds (DATA/STATE).

**Nothing DATA-like under a `*_CONFIG_HOME`** (decision 7): checked: pcm's `containers/` holds
definitions and `local.env`; `edges.yml` and `edges/` are the user's; volumes are in
`PCM_VOLUMES_HOME`. pmm's `machines/` holds definitions only. psm's `<name>.yml` are declarations.
`anfs/isos.yml` is the user's catalogue; the ISOs are in `~/.cache/anfs/isos`. The one list of
contributed content in a config dir is ruby's `default-gems` (finding 7). The reverse case
(firstmate's seeded `config/` under its DATA `home`) is an agent's sandboxed home and stays.

### RUNTIME

- **pmm's sockets** (`<name>.qmp`, `.ga`, build and test QMP): `${PMM_SOCK_DIR:-/tmp/pmm-<uid>}`
  on both systems. **In scope:** on Linux, when `XDG_RUNTIME_DIR` is set and a directory,
  `sock_dir_path` defaults to `$XDG_RUNTIME_DIR/pmm` (short paths, per-user, cleared at logout
  and boot, as qemu's sockets should be); otherwise, and on macOS, `/tmp/pmm-<uid>` as now
  (macOS has no `XDG_RUNTIME_DIR`, and its `$TMPDIR` is too long for a socket path). `PMM_SOCK_DIR`
  still wins (tests, `anfs work`). Under bats the live default is refused as today, for both
  defaults.
- **pmm's pid files** (`$PMM_STATE_HOME/run/<name>/pid` beside the port and host of the run):
  kept in STATE, as an exception the rule names: a pid stored with the record of the run it
  belongs to, guarded by `pid_alive`. Moving it would split one record across two places.
- **ppm's message file** (`core.sh:94`, `/tmp/ppm-messages.XXXXXX`) ignores `TMPDIR`: it becomes
  `${TMPDIR:-/tmp}`, so a job's `TMPDIR` (from `anfs work run`) holds it.
- Already right: the anfs RPC sockets (systemd's `%t`, i.e. `XDG_RUNTIME_DIR`), pcm's
  `toolkit_socket`, `mktemp "${TMPDIR:-/tmp}/…"` everywhere else.

## Implementation

### 1. The rule in core `AGENTS.md`

A section "Where files go (the XDG rule)" after "Repository Layout", replacing its one sentence:

- CONFIG — what the user declares: by hand, or by an explicit `set`/`add` command; never as a side
  effect of another command or an install. Includes each tool's shipped `<tool>.env` (linked from
  a package, which is a declaration in a repo) and the local definition directories (pcm's
  `containers/`, pmm's `machines/`: the `local` source, like Quadlet's `~/.config/containers/systemd`).
- STATE — what a tool records about what it did (trackers, records, backups, the run of a
  machine with its pid).
- DATA — content a tool manages (sources, volumes, disks, images).
- CACHE — what can be rebuilt (overrides, ISOs, update stamps).
- RUNTIME — sockets and pids that die with the session: `$XDG_RUNTIME_DIR` on Linux, `/tmp` on
  macOS (short paths), always overridable for tests.
- The accepted exceptions, each with its reason: platform locations (launchd, systemd units,
  podman's machine), stack/ssh's one `Include` line, a pid kept with its run record.
- How to check: the audit table above lives in this plan's log; a new write into a config
  directory in review is held to the rule.

`paths.sh`'s header and `AGENTS.md` "Key Files" agree with it.

### 2. The fixes (findings 2, 3, 4, 5, 6, 7, and RUNTIME)

As in the table. For 5: `pcm_state_env <id>` is `$PCM_STATE_HOME/containers/$(svc_name <id>)/state.env`;
`write_token` writes it through a temp file made 0600 (the directory 0700); `export_env_overrides`
exports `local.env` then `state.env`, each name only when unset, so the precedence is environment
> `local.env` > `state.env` > `.env.schema` defaults. If plan 04 has not merged, the user's file
is still `$PCM_ENV_HOME/<name>.env` and `state.env` is read below it the same way. `pcm show`
lists `state  <path>` when the file exists; `pcm remove <name>` deletes it (it is refetched on the
next `pcm up`); `pcm implode` already deletes `$PCM_STATE_HOME`. The cloudflared container's
docs and comments name the state file. For 6: `ssh_config_write` writes `$PMM_STATE_HOME/ssh_config` (same content);
the shipped `05-pmm.conf` has the one `Include`; `pmm implode` removes the state file and no
longer touches `~/.ssh` (ppm unlinks the shipped file on `ppm remove core/pmm`). For 7: the
`conf.d` setting and `install_gem`'s path; `ruby/install.sh` stops writing `~/.config/ruby`.

### 3. Recorded, not changed

Findings 1 (plan 01), 8, 9 (decision 14), 10, 11 (decision 13, `BACKLOG.md`), 12, 13: the log
carries each with its reason.

## Test Spec

- pmm (`tests/`, written, not run: pmm's suites are off; the log says so): `sock_dir_path` with
  `PMM_SOCK_DIR` set (it wins), with `XDG_RUNTIME_DIR` set on a stubbed Linux (`$XDG_RUNTIME_DIR/pmm`),
  unset on Linux and on macOS (`/tmp/pmm-<uid>`); under bats both live defaults are refused.
  `ssh_config_write` writes `$PMM_STATE_HOME/ssh_config` and nothing under the temp `~/.ssh`.
- pcm `edges.bats` (written, not run: pcm's suites are off): `write_token` writes
  `$PCM_STATE_HOME/containers/cloudflared/state.env`, mode 600, keeps other lines, and writes
  nothing under `$PCM_CONFIG_HOME`; a `TUNNEL_TOKEN` in `local.env` beats the state file, the
  environment beats both; `pcm remove` deletes the state file.
- `isolation.bats`: its socket pattern also catches `$XDG_RUNTIME_DIR/pmm` in a suite.
- ppm `tests/`: the message file lands under `$TMPDIR`.
- core: `ppm install` of core/pcm and core/pmm in a temp home (existing install tests, or
  `bash -n` plus a grep if none) creates nothing under `~/.config/pcm` or `~/.config/pmm` but
  linked files.
- stack ruby: `install_gem` appends to `$XDG_DATA_HOME/ruby/default-gems`; nothing under
  `~/.config/ruby` (its tests, if any; else a temp-home check by hand, in the log).
- A grep over core and stack for `mkdir -p .*CONFIG_HOME\|\.config/` in `install.sh` hooks finds
  only the accepted ones, listed in the log.

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing that
boots a VM or a box runs, nothing is installed on this host, and no machine or container starts.

- [ ] Re-run the audit against the job's trees (the same greps: every write into
      `$XDG_CONFIG_HOME`, `*_CONFIG_HOME`, `~/.config`, `~/.ssh`), and write the final table into
      `log.md`, each finding with what was done
- [ ] While working, the file you touch: `bats packages/anfs/tests/isolation.bats`,
      `bats packages/ppm/tests/<file>`; pmm's files: `bash -n` and `shellcheck` (its suites are off)
- [ ] `bats packages/anfs/tests`, `bats packages/ppm/tests`
- [ ] stack: `anfs work run <job> stack -- env ANFS_TEST_LIB=<job's core>/packages/anfs/home/.local/lib/anfs
      bats packages/ruby/tests`
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`): `ppm install -f core/pcm core/pmm stack/ruby`;
  if cloudflared runs here, delete the `TUNNEL_TOKEN=` line from its env file (`~/.config/pcm/env/cloudflared.env`
  or `containers/cloudflared/local.env`, whichever exists) and `pcm up cloudflared` (it refetches
  the token into `$PCM_STATE_HOME`; a line left in the user's file would win over it);
  `~/.ssh/config.d/05-pmm.conf` is now a link holding the `Include`, and `ssh <machine>` still
  works after `pmm up`; move `~/.config/ruby/default-gems` to `~/.local/share/ruby/default-gems`
  (or let the next `ppm install` write it) and delete the old one; `rmdir` the empty
  `~/.config/pmm/machines/{images,networks}` if you keep no definitions there. On a Linux host,
  stop every pmm machine before upgrading pmm (their sockets move to `$XDG_RUNTIME_DIR/pmm`).
  Run pmm's suites (`ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pmm`).
- Deferred to the end of the roadmap (named in `log.md`): `pmm`'s e2e on Debian (sockets under
  `$XDG_RUNTIME_DIR/pmm`, ssh through the state file), `roundtrip debian`, `roundtrip-macos`.
