---
---

# Plan 04 — A container's settings beside its definition: `containers/<name>/local.env`

Written 2026-10-21 against core main `9e8cb73` plus job `wsm-03` and linker plan 02, and plan
01 of this unit (merged first). Independent of plans 02 and 03.

## Context — read these files first

The job is `anfs work new <job> core stack`: core for pcm, its docs, `containers/ui` and the
`pcm-containers` skill; stack for its containers' docs and comments. Paths are core's unless they
name another repo.

- `units/config.md` — decisions 4, 6, 7; "Conflicts" (`pcm expose` does not exist; the tunnel
  token); decision 9 (the token to STATE, plan 05's); plan 01's `log.md`
- `packages/pcm/home/.local/bin/pcm` — the defaults block (`PCM_ENV_HOME`), the help's
  "Configuration" paragraph (lines ~112–116) and the `local` source line (~145)
- `packages/pcm/home/.local/lib/pcm/runtime.sh` — `export_env_overrides`, `in_service_dir`,
  `service_run`, `uses_fnox`
- `packages/pcm/home/.local/lib/pcm/ingress.sh` — `ingress_hosts` and its comment (the stale
  `pcm expose`)
- `packages/pcm/home/.local/lib/pcm/inspect.sh` — the `env` line of `pcm show`
- `packages/pcm/home/.local/lib/pcm/completion.sh` — `complete_service` (every directory, no marker)
- `packages/pcm/home/.local/lib/pcm/sources.sh` and `lib/anfs/catalog.sh` — `catalog_load …
  compose.yml` (the marker is already required), `catalog_dir`, the listing at line ~64
- `packages/pcm/home/.local/lib/pcm/edges/cloudflare-tunnel` — `write_token`; `edges.sh`
- `packages/pcm/home/.local/lib/pcm/implode.sh` (deletes `$PCM_CONFIG_HOME` whole: unchanged)
- `packages/pcm/install.sh` — `post_install`'s `mkdir` of `containers/` (plan 05's finding; see 4)
- `packages/pcm/tests/{helper.bash,edges.bats}` (`PCM_ENV_HOME`)
- Docs: `packages/pcm/README.md` (line 24), `AGENTS.md` "pcm: containers",
  `skills/pcm-containers/SKILL.md` (lines 34, 52), `containers/ui/README.md` (line 14); stack
  `containers/.gitignore`, `containers/cloudflared/{compose.yml,.env.schema,README.md}`,
  `containers/traefik/README.md`

## Overview

A container's settings for this machine are one file in the one directory the user already owns
for pcm: `~/.config/pcm/containers/<name>/local.env`. That directory is a definition when it has a
`compose.yml`, and settings only when it does not. `PCM_ENV_HOME` and `~/.config/pcm/env/` are gone.

## Implementation

### 1. Where a container's settings are

- `pcm_local_env <id>`: `$PCM_CONTAINERS_HOME/$(svc_name <id>)/local.env`. Keyed by the name, so
  it applies to whichever source's definition of that name wins, as `env/<name>.env` did.
- `PCM_ENV_HOME` goes from `bin/pcm`; nothing reads it (no shim, decision 6).
- `export_env_overrides` reads `pcm_local_env`; its own parser goes for plan 01's `anfs_conf_read` (the
  same rules, `export ` prefixes included). Precedence
  unchanged: environment > `local.env` > the definition's `.env.schema` defaults (varlock).
- A local definition's own directory may hold a `local.env` beside its `compose.yml`: the same
  file, the same meaning.

### 2. A directory is a definition only with `compose.yml`

- The catalogue already needs the marker (`catalog_load containers compose.yml`): a directory with
  only `local.env` is in no list. Checked by a test, not changed.
- `complete_service` skips a `$PCM_CONTAINERS_HOME/<name>/` without `compose.yml`. It completes a
  settings-only name through the sources that define it, as for any name.
- `pcm validate` and every command that takes a name: a settings-only directory is not a local
  definition; the name resolves through the sources (`local` does not shadow it).
- `pcm validate` warns about a settings-only directory whose name no source defines (`<name>:
  local.env for a container no source defines`), and about an empty directory under
  `containers/` (neither file).

### 3. Writers and readers

- `ingress_hosts`: reads `local.env`; its comment names it, and the `pcm expose` mention goes
  (there is no such command in main; if one lands later it writes `local.env`).
- `pcm show`: the `env` line becomes `settings  ~/.config/pcm/containers/<name>/local.env`, shown
  when the file exists, for a definition from any source.
- cloudflare-tunnel `write_token`: the token goes to STATE (decision 9), which is plan 05's. If
  plan 05 has merged, `write_token` already writes pcm's state file and this plan leaves it; if
  not, this plan points it at `pcm_local_env cloudflared` meanwhile (`mkdir -p`, mode 600, other
  lines kept, as now) so `PCM_ENV_HOME` can go, and plan 05 moves it on. The log says which.
- `bin/pcm`'s help: "Configuration" says overrides go in
  `$PCM_CONTAINERS_HOME/<name>/local.env`; the `local` source line says a directory there is a
  definition when it has `compose.yml`.

### 4. Not this plan

- `packages/pcm/install.sh`'s `mkdir -p "$pcm_config/containers"` (an empty directory on install)
  is plan 05's. If plan 05 has merged, it is gone; nothing here needs the directory to exist
  (`pcm_local_env`'s writer makes it).
- pmm's `~/.config/pmm/machines/` is unchanged (decision 7).

### 5. Documentation

Core `AGENTS.md` "pcm: containers": the local source, `local.env` (decision 4's text and
precedence), a settings-only directory. `packages/pcm/README.md` line 24. The `pcm-containers`
skill (`skills/pcm-containers/SKILL.md`): "Local overrides: `~/.config/pcm/containers/<name>/local.env`"
and the "Never ship a `.env`" paragraph. `containers/ui/README.md`. stack: `containers/.gitignore`'s
comment, cloudflared's `compose.yml` comment, `.env.schema` comment and README (the `echo … >>`
example becomes `~/.config/pcm/containers/twenty/local.env`), traefik's README.

## Test Spec

pcm's suites are off (roadmap, 2026-10-19): the tests are written and updated, and not run by the
agent; the log says so. They run when the person runs them
(`ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm`).

- `tests/helper.bash`: no `PCM_ENV_HOME`.
- `edges.bats`: the host tests write `$PCM_CONTAINERS_HOME/app/local.env`; `write_token` (when this plan
  is the one pointing it, above) writes `$PCM_CONTAINERS_HOME/cloudflared/local.env`, keeps
  `OTHER=kept`, mode 600.
- `sources.bats` (or a new `local-env.bats`): a directory with only `local.env` is not in `pcm
  list` and does not shadow a source's definition of the name; `complete_service` leaves it out
  and still offers the name from the source; `pcm show <name>` prints the `settings` line;
  `local.env` beats `.env.schema`'s default and the environment beats `local.env` (a stub varlock
  or `export_env_overrides` directly); a local definition with `compose.yml` and `local.env` uses
  both; `pcm validate` warns about a settings-only directory no source defines, and an empty one.
- `grep -rn 'PCM_ENV_HOME\|pcm/env'` in core and stack: nothing.

## Verification

Every command runs in the job: `anfs work run <job> <repo> -- <cmd> </dev/null`. Nothing that boots
a VM or a box runs, nothing is installed on this host, and no container is started.

- [ ] While working: `bash -n` and `shellcheck` on each pcm file you touch (its bats files are
      off: do not set `ANFS_TESTS_PMM_PCM`)
- [ ] `bats packages/anfs/tests` (catalogue, registry)
- [ ] `grep -rn 'PCM_ENV_HOME\|pcm/env'` in core and stack finds nothing
- [ ] Once, at the end: `anfs work run <job> core -- packages/anfs/tests/all`
- Not run, for the person after merge (in `log.md`): `ppm install -f core/pcm`; move
  `~/.config/pcm/env/ui.env` to `~/.config/pcm/containers/ui/local.env`, then `rmdir
  ~/.config/pcm/env`; `pcm show ui` names it and `pcm up ui` keeps listening where it did. The
  eight empty directories under `~/.config/pcm/containers/` (authentik, dnsmasq, dockge,
  glitchtip, n8n, netboot, postgres, telemetry; empty since 2026-10-01) are neither definitions
  nor settings: `rmdir` them (`pcm validate` now warns about them). Run pcm's suites
  (`ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm`).
- Deferred to the end of the roadmap (named in `log.md`): `roundtrip debian` (pcm's part).
