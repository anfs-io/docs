---
---

# Plan 02 — `pcm provision` / `pcm deprovision`: a service's hooks for a consumer outside pcm

Written 2026-10-22 against core main `2cbc7f9`, in the job `pcm-provision` (core and docs), for
report r-20261022-2 (it blocks the cnfs session: partitia's `pt services` is the first consumer).
The person's decisions of 2026-10-22 are in the job's brief and below; the choices this plan makes
are marked **Decided here**.

## Context — read these files first

- r-20261022-2 (`BACKLOG.md` of the live docs; not yet committed there, so not in this job's copy)
- `lib/pcm/deps.sh` (`has_hook`, `run_hook`, `provision`, `start_deps`), `lib/pcm/remove.sh`
  (`remove_summary`, deprovision in `cmd_remove`), `lib/pcm/compose.sh` (`compose_command up`),
  `lib/pcm/runtime.sh` (`service_run`, `wait_healthy`, `project_containers`)
- `lib/pcm/commands.sh`, `bin/pcm`'s epilogue, `packages/pcm/README.md`, `skills/pcm-containers`
- `packages/pcm/tests/run.bats` (podman and the compose runner stubbed)

## The commands

```
pcm provision   <service> [--as NAME] [key=value...]     # --json: one object
pcm deprovision <service> [--as NAME] [key=value...]
```

- **provision**: no hook → exit 1 (`<id> has no provision hook`). Not running → started the way
  `pcm up <service>` starts it (`compose_command up`: validation, memory and ports, edges, its own
  `x-pcm.depends_on`), its output on stderr; running from another definition → refused
  (`check_not_running_elsewhere`). Then `wait_healthy` (`PCM_HEALTH_TIMEOUT`), then the hook,
  through the same code `up` uses for a dependent. stdout is only the hook's `KEY=VALUE` lines
  (the filter `provision()` applies); `--json` prints them as one object (`anfs_result`), keys as
  the hook prints them (`URL`, not `PCM_POSTGRES_URL`: the consumer names its own variables).
- **deprovision**: no hook → exit 1. Not running → nothing is started; the message is remove's
  (`<id> is not running; what it provisioned for NAME is left in place`), exit 1. Running → the
  hook, its output on stderr, as `remove` runs it.
- **`--as NAME`** is `PCM_DEPENDENT` (default `external`). **Decided here:** a name of letters,
  digits and `._/:@-` (`partitia/shop` fits); anything else is a usage error.
- **key=value args**: dependents already have the mechanism, their `depends_on` map, passed to the
  hook as `key=value` arguments; these are passed the same way, as given. Anything else is a usage
  error (**Decided here**: `^[A-Za-z_][A-Za-z0-9_.-]*=`), so a typo does not reach a hook.
- **Secrets**: hook output is never logged. The only log lines are anfs_warn's, which name the
  keys (`provisioned stack/postgres for smoke-test: URL HOST ...`), never a value. The args are
  the command line, which `anfs_log_start` records as for every command: documented as "settings,
  not secrets".

## Reuse, not a copy (`deps.sh`)

- `run_hook` splits: `hook_run <hook> <dep> <dependent-name> [settings...]` runs the hook (env:
  `PCM_SERVICE`, `PCM_PROJECT`, `PCM_DEPENDENT`; `service_run --no-redact-stdout`, so varlock,
  `local.env`, fnox); `run_hook` keeps its signature and calls it with `dep_settings`.
- `hook_pairs`: the `KEY=VALUE` filter, used by `provision()` and by `pcm provision`.
- The commands live in a new `lib/pcm/provision.sh`; `commands.sh` declares them after `remove`.

## Docs and version

Help (`commands.sh`, a paragraph in `bin/pcm`'s epilogue), completion (the registry: `service`,
then nothing), README section "Provisioning for a consumer outside pcm", the same in
`skills/pcm-containers` (a host-run Rails app), core `AGENTS.md` one bullet; pcm 0.3.0.

## Test Spec (`packages/pcm/tests/provision.bats`; podman, compose and varlock stubbed)

provision when running (no compose call; env and args the hook sees; only KEY=VALUE on stdout);
when stopped (compose `up -d` first); no hook (exit 1, nothing started); `--as`; `--json`;
key=value args, and a bad arg / bad `--as` (64); running from another definition (refused);
deprovision running (hook output on stderr) and stopped (left in place, exit 1, no hook run, no
compose); a hook that fails (exit 1, nothing on stdout); values never in `ANFS_LOG`'s file, keys
are. `up`'s provisioning is unchanged (the existing dependents' tests, and one here through
`start_deps`).

## Verification

- `ANFS_TESTS_PMM_PCM=1 bats packages/pcm/tests` before and after (old failures: manifest.bats 55,
  toolkit.bats 83); `packages/anfs/tests/all` once at the end.
- By hand, after merge (in `log.md`): `ppm install -f core/pcm`, then `pcm provision postgres --as
  smoke-test database=smoke_test` and `pcm deprovision postgres --as smoke-test database=smoke_test`.
