---
plan: config/04-pcm-local-env
job: config-04
date: 2026-10-22
---

# Log: config/04-pcm-local-env

## The plan against the job's base

The job's core and stack start at config-01's tip (core `533935e`, stack `2f989ee`): the loader
(`anfs_conf_read`) and `pcm.env` are there. **Plan 05 has not merged**, so `write_token` is pointed
at `local.env` meanwhile (plan, section 3); plan 05 moves it to STATE. `install.sh`'s `mkdir` of
`containers/` is plan 05's and is untouched.

## What was built

core, pcm:
- `pcm_local_env <id>` (`runtime.sh`): `$PCM_CONTAINERS_HOME/<name>/local.env`, keyed by the name.
- `export_env_overrides` reads it through `anfs_conf_read` (comments, `export `, one pair of quotes
  as in `<tool>.env`; its own parser is gone). Below the environment, as before.
- `PCM_ENV_HOME` gone from `bin/pcm` (no shim).
- `complete_service`: a directory without `compose.yml` is skipped (local and sources alike; the
  catalogue already needed the marker, unchanged).
- `check_local_dirs` (`validate.sh`), run first by `pcm validate` (with or without names):
  `<name>: warning: local.env for a container no source defines`, and `<name>: warning:
  ~/.config/pcm/containers/<name> has neither compose.yml nor local.env`. Warnings only.
- `pcm show`: `settings  ~/.config/pcm/containers/<name>/local.env` when the file exists (was `env`).
- `ingress_hosts`' comment names `local.env`; the `pcm expose` mention is gone.
- cloudflare-tunnel `write_token`: `$PCM_CONTAINERS_HOME/cloudflared/local.env` (`mkdir -p`, mode
  600, other lines kept). This also fixes a latent bug: the edge is its own process and
  `PCM_ENV_HOME` was never exported by `bin/pcm`, so under `set -u` a real `add` would have died
  on it; `PCM_CONTAINERS_HOME` is exported.
- Help: "Configuration" names `$PCM_CONTAINERS_HOME/<name>/local.env`; the `local` source line says
  a directory without `compose.yml` is settings only.

Docs: core `AGENTS.md` "pcm: containers" (one bullet: the local source, `local.env`, precedence, a
settings-only directory, show/validate, the token stopgap); `packages/pcm/README.md`;
`containers/ui/README.md`; the `pcm-containers` skill (local source needs `compose.yml`; local
overrides; "Never ship a `.env`"). stack: `containers/.gitignore`, cloudflared's `compose.yml`,
`.env.schema` and README (the hosts example is now `mkdir -p …/twenty; echo … >>
…/twenty/local.env`), traefik's README.

## Commits (branch config-04, not pushed)

core:
- `2791dc9` pcm: `local.env`, `PCM_ENV_HOME` gone, completion, validate, show, the edge; tests
- `afe8ef4` docs (AGENTS.md, pcm README, ui README, pcm-containers skill)

stack:
- `4a5fcaa` containers' docs and comments

Versions (pre-commit hook): `containers/ui` 0.1.3; stack authentik, cloudflared, dockge, glitchtip,
n8n, traefik 0.1.1. `packages/pcm` (0.1.10) and `packages/anfs` (0.2.15) already carry an unpushed
bump (linker/02, config-01), so the hook left them (one bump per unpushed series).

## Decisions made during the work

- **No `~` expansion in `local.env`**: values are a container's environment (compose's `.env`
  semantics), so `export_env_overrides` uses `anfs_conf_read` as written, not `anfs_load_conf`'s
  `anfs_conf_expand`. Documented in AGENTS.md and the function's comment.
- **`complete_service` requires `compose.yml` in the sources too**, not only in the local
  directory: one rule, the catalogue's.
- **`pcm validate` checks the local directories on every run**, named services or not (cheap; the
  warnings are about the machine, not a service).
- **The tests that run** live in `packages/anfs/tests/pcm-local-env.bats`. `packages/anfs/tests/all`
  leaves out every file under `packages/pcm/tests/` and pcm's helper skips in `setup_file`, so a
  pcm file cannot run without turning the suite on. This file loads pcm's helper (`pcm_setup`,
  `define`, `add_source`), resets `PCM_PKG`, and redefines `setup_file` as a no-op; it needs no
  podman and runs in 8 s. The plan's `local-env.bats` cases are all here (not duplicated in pcm's
  off suite). The `write_token` case uses its own few-line curl stub (`edges.bats`'s
  `fake_cloudflare` is local to that file).

## Tests (all through `anfs work run config-04 core -- … </dev/null`)

- `bash -n` (bash 5) on `bin/pcm`, `runtime.sh`, `ingress.sh`, `inspect.sh`, `completion.sh`,
  `validate.sh`, `edges/cloudflare-tunnel`: clean.
- `pcm-local-env.bats` **10/10** (new): `pcm_local_env`; a settings-only directory is not in
  `all_ids`/`pcm list` and does not shadow `a/app`; `complete_service` leaves it out and offers
  `app`/`a/app`; `export_env_overrides` with `export`, quotes, comments, the environment (empty
  included) winning; **real varlock** (temp `HOME`): `local.env` > `.env.schema` default, the
  environment > `local.env`, the schema default without the file; a local definition with
  `compose.yml` and `local.env` (`canon` → `local/app`, both used); `pcm show` prints the
  `settings` line, and not without the file; `pcm validate` warns for a settings-only name no
  source defines and an empty directory, and not for `app`; cloudflare-tunnel `add` writes
  `TUNNEL_TOKEN` into `cloudflared/local.env`, keeps `OTHER=kept`, replaces the old token, mode 600,
  and makes the directory when missing; `grep` for `PCM_ENV_HOME|pcm/env` in core's packages,
  containers, skills, AGENTS.md, README.md: nothing.
- `settings.bats`, `catalog.bats`, `registry.bats`, `isolation.bats`: pass (with the above, 70/70).
- By hand: a temp `HOME` under the job's `TMPDIR` with the job's `lib/anfs` and `lib/pcm` linked,
  `env -i`: `pcm help` shows the new Configuration and Sources text; `pcm validate` with
  `containers/{ghost,ui}/local.env` and an empty `containers/empty/` prints the three warnings.
- `grep -rn 'PCM_ENV_HOME\|pcm/env'` in core and stack: only the test that checks for it.
- **Once, at the end**: `packages/anfs/tests/all`: **538 passed, 0 failed**, 38 files (pmm and pcm
  off), 59 s.

## Not run

- **pcm's suites** (`packages/pcm/tests`), off by the roadmap (2026-10-19); `ANFS_TESTS_PMM_PCM`
  was not set. Changed there and not run: `helper.bash` (no `PCM_ENV_HOME`), `edges.bats` (the
  host tests write `$PCM_CONTAINERS_HOME/app/local.env`; the token test reads
  `$PCM_CONTAINERS_HOME/cloudflared/local.env`). Run by the person:
  `ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm`.
- `shellcheck`: not installed on this host (not in brew, not in mise); `bash -n` only.
- stack's suites: only container docs and comments changed (no code).
- Deferred to the end of the roadmap: `roundtrip debian` (pcm's part).
- Nothing installed; no container started; `~/.config/pcm` only read.

## Deviations

- stack's authentik, dockge, glitchtip and n8n READMEs said "add a `.env` next to it (e.g. stowed
  from your own ppm layer)", which contradicts "never ship a `.env`" and this plan; each now names
  `~/.config/pcm/containers/<name>/local.env`. Not in the plan's file list (docs only).
- The plan's `local-env.bats` is `packages/anfs/tests/pcm-local-env.bats` (above), so it runs.
- `settings.bats`'s pcm allow-list drops `PCM_ENV_HOME` (the variable is gone).

## Shared files touched (for config-02/03's rebase)

core: `AGENTS.md` (one bullet added in "pcm: containers"), `packages/anfs/tests/settings.bats`
(one word in the pcm allow-list line), `packages/pcm/home/.local/bin/pcm` (the `PCM_ENV_HOME`
default line and two help paragraphs; not the header or the sourcing block),
`packages/pcm/tests/helper.bash` (one export line). New: `packages/anfs/tests/pcm-local-env.bats`.
Everything else is pcm-only or a container's docs.

## Open questions

1. **`~` in `local.env`** is kept literal (decision above). Say if it should expand like the
   tools' `.env` files.
2. **stack's `containers/.gitignore`** ignores `.env`, `.env.local`, `*.local.*`, but not
   `local.env`; nothing reads a `local.env` inside a clone, so it was left. Add it?

## By hand, for the person after merge

1. Pull core (`anfs src update`) and reinstall pcm: `ppm install -f core/pcm`.
2. Move the one settings file and drop the old directory (read 2026-10-22: `env/` holds only
   `ui.env`, with `ANFS_UI_BIND`, `ANFS_UI_HOSTS`, `ANFS_UI_REMOTE_ACTIONS`):

   ```bash
   ls -A ~/.config/pcm/env                       # expect: ui.env
   mkdir -p ~/.config/pcm/containers/ui
   mv ~/.config/pcm/env/ui.env ~/.config/pcm/containers/ui/local.env
   rmdir ~/.config/pcm/env
   ```

3. The eight empty directories under `~/.config/pcm/containers/` (read 2026-10-22, each empty,
   dated 2026-10-01): neither definitions nor settings, and `pcm validate` now warns about them:

   ```bash
   cd ~/.config/pcm/containers && rmdir authentik dnsmasq dockge glitchtip n8n netboot postgres telemetry
   ls -A ~/.config/pcm/containers                 # expect: ui
   ```

4. Check: `pcm show ui` prints `settings  ~/.config/pcm/containers/ui/local.env`; `pcm validate`
   has no `warning` about `containers/`; `pcm up ui` keeps listening where it did
   (`ANFS_UI_BIND`/`ANFS_UI_HOSTS` from the moved file).
5. Run pcm's suites: `ANFS_TESTS_PMM_PCM=1 packages/anfs/tests/all pcm`.
6. If `PCM_ENV_HOME` is set anywhere (shell rc, fnox, `.envrc`), drop it; nothing reads it:
   `grep -rn PCM_ENV_HOME ~/.zshrc ~/.zshenv ~/.config/sh ~/.config/zsh ~/.config/fnox 2>/dev/null`.
