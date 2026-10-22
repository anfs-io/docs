---
plan: pcm/01-run
job: pcm-run
date: 2026-10-22
---

# Log: pcm/01-run

## The plan against the job's base

core and docs start at main (core `78aac47`, docs `861d16c`). The compose provider on this Mac is
podman-compose 1.6.0 (podman 6.1.1); its `run` takes `--name`, `--no-deps`, `--rm`, `-u`, `-T`,
`-l`, as docker compose's does. podman 6's `ps --filter label!=<key>` excludes and ANDs with
`label=`: checked with read-only `podman ps` on this Mac. The fixed `pcm show` format was checked
the same way, with `podman inspect` on an existing pcm container (`telemetry_grafana_1`): it
prints the mounts, and the old one fails with `can't evaluate field Name in type []interface {}`.

## What was built

core, pcm (`lib/pcm/run.sh` new; `runtime.sh`, `deps.sh`, `box.sh`, `inspect.sh`, `remove.sh`,
`commands.sh`, `bin/pcm`):
- `pcm run <service> [-s NAME] [-u USER] [--keep] [--] [cmd...]`, `--passthrough`: compose `run
  --name <name>-run-<6 hex> --no-deps [--rm] [-u USER] [-T] <compose-service> [cmd...]` through
  `compose_run` (varlock, `local.env`, `state.env`, fnox, the toolkit token). `-T` unless stdin
  and stdout are terminals. The command's exit status. `--keep` says the name and how to delete it.
- Before it, as `up`: `pcm_podman_ready`, `validate_services` and `check_memory` over the service
  and its declared dependencies' closure, `check_not_running_elsewhere`, privileges announced,
  `start_deps <id> "" declared`, `ensure_mount_dirs`; then the compose service's own
  `depends_on` siblings not running are started with `up -d` through the service's ordinary
  override and waited for.
- The one-off's override (`ONEOFF=true compose_flags`): `$PCM_CACHE_HOME/<name>.run.override.yml`,
  pcm's labels plus `io.pcm.oneoff: "true"`, no ingress labels, no network alias; the rest as
  `up`'s.
- `declared_deps` (only `x-pcm.depends_on`), which `service_deps` now starts from;
  `start_deps`' third argument `declared`.
- `label!=io.pcm.oneoff` in `project_containers`, `_box_container`, snapshot's commit lookup,
  `pcm_containers` and `pcm show`'s container table. `oneoff_containers` lists them.
- `pcm remove`: `podman rm -f` of the one-offs before compose `down`, a `one-offs` line in the
  summary, the run override deleted. `pcm show`: a `one-offs` line. `implode` unchanged (it
  already deletes every container with `io.pcm.name`).
- `pcm show` on podman 6: `PCM_SHOW_MOUNTS_FORMAT='{{$n := .Name}}{{range .Mounts}}{{$n}}…'`.
- Help: `run` in the registry (after `exec`); the epilogue's "One-offs" paragraph, with
  `podman run --rm <image>` for a stock image. Completion comes from the registry (`service`).
- `packages/pcm/package.yml` 0.2.0 (by hand; the hook left it).

Docs: core `AGENTS.md` "pcm: containers" (a One-offs bullet), `packages/pcm/README.md` ("One-off
commands"), `skills/pcm-containers` ("Snapshots, run, exec and shell", and plain `podman run --rm`
for a stock image). docs: `units/pcm.md` (new), this plan and log, the ROADMAP's Last item on
`pcm show` ticked.

## Commits (branch pcm-run, not pushed)

core:
- `254b034` pcm run, the one-off label and its filters, remove and show, the podman 6 show fix,
  help, 0.2.0; `tests/run.bats`
- `cb4facd` docs (AGENTS.md, pcm README, pcm-containers skill)
- `17d2ec0` the one-off switch renamed `ONEOFF` (was `PCM_ONEOFF`, which settings.bats took for an
  undocumented setting)

docs:
- `47cb780` `units/pcm.md`, the plan
- this log and the ROADMAP tick

## Decisions made during the work

- **Dependencies**: `run` starts what the definition declares required, as `up` does:
  `x-pcm.depends_on` (started, waited for, provisioned, so the one-off has `PCM_<DEP>_*`) and the
  compose service's own `depends_on`. Not the implicit proxy and edges of an ingress service: a
  one-off gets no route. Nothing started is stopped on exit.
- **Siblings through the ordinary override, the one-off `--no-deps`**: letting compose start them
  from the one-off's override would give them no ingress labels.
- **Several compose services and no `-s`** is a usage error naming them (`exec` picks the first
  running container; `run` has none to pick).
- **No command** runs what the compose service runs (`command:`, else the image's default), as
  compose run does.
- **The name** `<name>-run-<hex>`, so `pcm ps` (unchanged: all of pcm's containers) tells a
  one-off apart without a new column.
- **The memory check** leaves one-offs out of what runs (`pcm_containers`): a kept one-off's
  `memory:` is not counted.
- **`ONEOFF`, not `PCM_ONEOFF`**: an internal global like `CLOSURE`, not a setting (deviation
  from the plan's name).

## Tests (all through `anfs work run pcm-run core -- … </dev/null`)

pcm's suite was run for this job at the person's request, with `ANFS_TESTS_PMM_PCM=1` per
command (its helper skips every test without it).

- **Before**, on the base commit `78aac47`: `bats packages/pcm/tests`: **95/97**, 2 failing:
  - `manifest.bats` 55 "machine resize: running containers and no terminal, without -y": expects
    status 1, gets 77 (`anfs_confirm` without a terminal is 77 since setup/01). Test drift.
  - `toolkit.bats` 83 "macOS: compose runs with the grant's token in its environment": the
    test's `RUNNER=(showenv)` is a shell function, and `in_service_dir` `exec`s it (127). Test
    drift; the compose runner must be a file (run.bats writes one).
- **After**: **108/110**, the same two failing, nothing new. `run.bats` **13/13** (new); a
  mutation check (the old `inspect.sh`) fails its three show/filter cases.
- `bash -n` on every pcm lib: clean. `pcm help run` and `pcm __complete run …` checked with the
  job's libraries (`PCM_LIB_DIR`, `ANFS_LOG=off`).
- **Full run** (`packages/anfs/tests/all`, pmm and pcm off): first **620 passed, 2 failed**
  (settings.bats: `PCM_ONEOFF`, fixed in `17d2ec0`; registry.bats' "a tab press under 100 ms":
  113 ms under the parallel load). After the fix: **621 passed, 1 failed**, 46 files, 124 s: only
  the registry timing (122 ms), which passes alone (`bats settings.bats registry.bats`: 50/50); it
  times a test tool, not pcm.

## Not run

- No real container, machine or image was created, started or removed: podman and the compose
  runner are stubbed. `podman ps`/`podman inspect` were run read-only.
- `shellcheck`: not installed here.
- Deferred to the end of the roadmap: `roundtrip debian` (pcm's part).

## Open questions

- The two pre-existing failures in pcm's suite (above) are test drift with one-line fixes
  (`[ "$status" -eq 77 ]`; a file runner in toolkit.bats). Not fixed here: outside the plan. A
  follow-up, or this branch, if the person wants.

## By hand, for the person after merge

1. `anfs src update`, then `ppm install -f core/pcm` (pcm 0.2.0 in `ppm list`).
2. The `pcm show` fix, on a running service: `pcm show ui` ends with a `mounts` section, no
   template error.
3. A one-off, removed on exit:

   ```bash
   pcm run postgres -- psql --version        # prints the version, exit 0
   pcm ps -a | grep -- -run- || echo none    # none
   ```

4. With a dependency's env and network (starts postgres if it is not running, and provisions
   n8n's database as `pcm up n8n` would; n8n itself is not started):

   ```bash
   pcm run n8n -s n8n -- sh -c 'env | grep ^DB_ | cut -d= -f1'   # the DB_* names n8n gets
   ```

5. Kept, then seen and deleted (not with `pcm remove postgres`, which deletes its data too):

   ```bash
   pcm run --keep postgres -- true           # pcm: kept postgres-run-xxxxxx (...)
   pcm ps -a | grep postgres-run-            # there
   pcm show postgres | grep one-offs         # named
   pcm list | grep postgres                  # STATUS as before: the one-off is not the service
   podman rm postgres-run-xxxxxx
   ```

6. At a terminal, a tty: `pcm run postgres -- bash`, then `exit`.
7. `pcm down postgres` if step 4 started it. Run pcm's suites:
   `ANFS_TESTS_PMM_PCM=1 bats packages/pcm/tests` (expect the two failures above until fixed).
