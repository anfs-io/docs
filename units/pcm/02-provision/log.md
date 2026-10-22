---
---

# Log 02 — `pcm provision` / `pcm deprovision`

2026-10-22, job `pcm-provision`, for r-20261022-2. core `0c28976` on `2cbc7f9`, pcm 0.3.0.

## What was done

- `lib/pcm/provision.sh` (new): `cmd_provision`, `cmd_deprovision` and their shared argument checks
  (the service, `--as`, the `key=value` settings, the hook).
- `lib/pcm/deps.sh`: `run_hook` is split. `hook_run <hook> <dep> <dependent-name> [settings...]`
  runs a hook (env, varlock, `--no-redact-stdout`), and `run_hook` keeps its signature for `up`,
  `remove` and the proxy's labels hook. `hook_pairs` is the `KEY=VALUE` filter `provision()` used
  inline, and `pcm provision` uses it too. Nothing about `up`'s provisioning changed (a test pins
  it).
- `commands.sh` declares both commands after `remove` (help and completion come from there). There
  is a paragraph in `bin/pcm`'s epilogue, a README section, a skill section ("Provisioning for a
  consumer outside pcm") and a bullet in core `AGENTS.md`. pcm is 0.3.0.

## Decisions taken here

- Starting a stopped service is `compose_command up`, i.e. `pcm up` itself (validation, memory,
  ports, edges, its own dependencies). Its output goes to stderr. A service running from another
  definition is refused, as `start_deps` refuses it.
- Dependents had an args mechanism already (the `depends_on` map, passed as `key=value`
  arguments), so the command's args are passed the same way. Anything that is not
  `key=value` is a usage error (64). `--as` is letters, digits and `._/:@-`.
- The `--json` keys are the hook's own (`URL`), not `PCM_<DEP>_*`. The consumer names its own
  variables.
- A provision hook that fails prints nothing on stdout (no partial set) and exits 1.
- deprovision does not wait for health (remove doesn't either). Its hook's stdout goes to stderr,
  as in remove.
- The log: `pcm provision` logs `provisioned <id> for <name>: <KEY> <KEY>...`. The values are not
  logged. **The args are logged**: `anfs_log_start` records every command line, so the docs say
  the settings are not secrets.

## Tests

- `packages/pcm/tests/provision.bats` (13 tests; podman, the compose runner and varlock stubbed)
  covers:
  - provision when running and when stopped (`up -d` first)
  - `--json`, `--as`, the args, and usage errors
  - no hook, a failing hook, another definition running
  - deprovision when running and when stopped
  - values never in the log
  - `start_deps` unchanged
  - completion
- `ANFS_TESTS_PMM_PCM=1 bats packages/pcm/tests`: 108/110 before, 121/123 after. The 2 failures are
  the old ones (manifest.bats "machine resize ... without -y"; toolkit.bats line 83, "macOS: compose
  runs with the grant's token").
- `packages/anfs/tests/all`: 662/662.

## Notes

- r-20261022-2 is only in the live docs' `BACKLOG.md`, which is uncommitted there. This job's
  copy doesn't have it.
- postgres's provision hook prints the container name as `HOST` and in `URL`. That name resolves
  only on the shared network, so a host-run consumer has to use `localhost:$POSTGRES_PORT`. The
  skill says so. If partitia wants a host-usable URL from the hook, that is a change in stack's
  postgres (e.g. a `HOST_URL` key), not in pcm.

## For the person, after merge

1. `ppm install -f core/pcm`, then `pcm --version` (0.3.0) and `pcm help provision`.
2. Smoke test against stack's postgres, which has both hooks (valkey and traefik have only
   provision):

   ```sh
   pcm provision postgres --as smoke-test database=smoke_test          # URL=... HOST=... DATABASE=smoke_test
   pcm provision postgres --as smoke-test database=smoke_test --json   # the same, one object
   grep smoke-test ~/.local/state/anfs/anfs.log | tail -3              # key names, no password
   pcm deprovision postgres --as smoke-test database=smoke_test        # "dropped database 'smoke_test'"
   pcm provision valkey --as smoke-test                                # provision only: prints its pairs
   pcm deprovision valkey                                              # exit 1: no deprovision hook
   ```

   With postgres stopped first, `pcm provision` starts it. `pcm down postgres` followed by
   `pcm deprovision postgres --as smoke-test database=smoke_test` says it is left in place and exits 1.
3. Close the report: `anfs report close r-20261022-2 --commit <merged sha> --source core`, and
   tell the cnfs session.
