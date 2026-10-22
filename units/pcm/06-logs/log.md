---
unit: pcm/06-logs
job: pcm-logs
date: 2026-10-22
report: r-20261022-6
---

What: `pcm logs <service> [-s NAME] [-f|--follow] [--tail N] [--since T] [-- compose-logs-options]`, a plain command (lib/pcm/logs.sh); pcm 0.4.0.
Why: logs were only `podman compose --pcm logs`, a zsh function in pcm.zsh, so bash, scripts and agents could not get them.
How: `compose_flags` + `compose_run ... logs` (as up runs compose); the zsh wrapper is kept (other compose subcommands) and its logs path is `cli_call logs` in `compose_wrapper`.

# Log: pcm/06-logs

## Decisions

- Through compose logs, not `podman logs` over the containers: the project, compose files and env
  are resolved the way up does, and with no `-s` compose shows every compose service. podman-compose
  logs reads the containers the compose file defines, so `pcm run`'s one-offs are left out; `-s
  <name>-run-<hex>` (a one-off, from `pcm ps`) is `podman logs` on that container.
- No containers (stopped ones count: their logs are still there) is exit 1, "has no containers
  (pcm up <name>)"; with `-s`, "of compose service X". An unknown `-s` is 64, listing the services.
- When the containers came from another source's definition of the name (io.pcm.id), that
  definition's compose file is used.
- `podman compose --pcm` stays (up, down, restart, ...). Its logs takes one pcm service now (64
  otherwise): compose_command ran the services one after another, so `-f a b` never reached b.
  Global compose options before `logs` are ignored with a warning.
- As every compose call, logs regenerates `<name>.override.yml` (same content as up wrote).

## Tests

- tests/logs.bats (new): all compose services; `-s`; `-f`/`--follow`, `--tail`, `--since` and
  `-- -t` passed on; no containers (whole service and one compose service); unknown service and
  compose service (64); one-offs left out and shown when named; the --pcm wrapper; bin/pcm under
  `env -i ... bash -c 'pcm logs ui --tail 5'` with stub podman and compose.
- pcm's bats with `ANFS_TESTS_PMM_PCM=1`: 139, all pass but the known manifest.bats "machine
  resize" and toolkit.bats "macOS: compose runs with the grant's token" failures.
- `packages/anfs/tests/all`: 662 passed, 0 failed (pmm and pcm off).

## By hand, after merge

Reinstall core/pcm (0.4.0), with core/ui running:

    bash -c 'pcm logs ui --tail 5'
    bash -c 'pcm logs ui -f'            # follows; ctrl-c
    pcm logs ui -s nope; echo $?        # 64, lists ui's compose services

The first prints the last 5 lines of each ui container; nothing sources pcm.zsh. Close
r-20261022-6 in BACKLOG.md.
