---
---

# Plan 01 — `pcm run <service> [--] <cmd...>`: a one-off container from a service's definition

Written 2026-10-22 against core main `78aac47` (config/05 and linker/05 merged), in the job
`pcm-run` (core and docs). The person's decisions of 2026-10-22 are in the job's brief and
below; the choices this plan makes are marked **Decided here**.

## Context — read these files first

- `ROADMAP.md`: "For an agent running this list", the Next item `pcm/01-run`, the Last item on
  `pcm show` (podman 6)
- `units/container-images.md` and its plan 01: they proposed `pcm run <image>`; the person
  decided against it (below), so this plan settles that part
- `packages/pcm/home/.local/bin/pcm` (the help epilogue), `lib/pcm/commands.sh` (the registry)
- `lib/pcm/compose.sh` (`compose_command`: what `up` does before compose), `lib/pcm/runtime.sh`
  (`compose_flags`, the override, `compose_run`, `service_run`, `export_env_overrides`,
  `project_containers`, `wait_healthy`), `lib/pcm/deps.sh` (`service_deps`, `start_deps`,
  `dep_closure`), `lib/pcm/box.sh` (`exec`, `shell`, `_box_container`, snapshot's lookup),
  `lib/pcm/inspect.sh` (`pcm_containers`, `cmd_show`), `lib/pcm/remove.sh`, `lib/pcm/implode.sh`,
  `lib/pcm/ingress.sh` (`ingress_labels`)
- `packages/pmm/home/.local/lib/pmm/commands.sh` (`run`'s wording), `skills/pcm-containers`
- `packages/pcm/tests/helper.bash` (podman and the compose runner stubbed)

## Overview

`pcm run <service> [options] [--] [cmd...]` is `podman compose run` with everything `pcm up`
gives a service: its compose.yml, pcm's override (labels, the shared network, mount sets, the
toolkit grant, `memory:`, the active snapshot's image), its env through varlock (the schema,
`local.env`, `state.env`, fnox, what its dependencies' provision hooks print). The container is
marked `io.pcm.oneoff=true`, so the commands that mean "the service's own containers" leave it
out and `remove`/`implode` delete it. It is removed on exit unless `--keep`.

There is no `pcm run <image>`: a throwaway run of a stock image is `podman run --rm <image>`, and
pcm, its help, its README and `skills/pcm-containers` say so, so an agent doing that does not
think it bypasses pcm.

## Implementation

### 1. The command (`lib/pcm/run.sh`, registered in `commands.sh`)

```
pcm run <service> [-s NAME] [-u USER] [--keep] [--] [cmd...]
```

- `--passthrough`, as `exec`: options pcm does not declare are part of the command
  (`pcm run app ls -la`); `--` ends pcm's own (`pcm run app -- psql -U x -s y`).
- `-s --service NAME`: the compose service. **Decided here:** with none, the definition's only
  compose service; with several, a usage error naming them (a one-off in the wrong compose
  service is worse than a question; `exec` picks the first running container, which `run` has
  none of).
- `-u --user USER`: compose run's `-u`.
- `--keep`: no `--rm`. The container's name is printed on stderr with how to remove it.
- No command: compose run with none, so what the compose service runs: its `command:`, else the
  image's default (**Decided here**, the brief's "the image's default command" read as compose
  does, since a definition's `command:` is part of the service).
- A tty (`-T` left off) when stdin and stdout are terminals, as `exec` decides; `-T` otherwise.
  stdin is always attached (both compose providers run one-offs interactive).
- The exit status is the command's.
- The name: `<name>-run-<6 hex>` (`--name`), so `pcm ps` and `podman ps` tell it from the
  service's own `<name>_<svc>_1`.

### 2. What runs first, following `up`

- `pcm_podman_ready`; the definition and its dependencies validated (`validate_services` over
  the closure below), `check_memory` over the same closure, `export_podman_socket`;
  `check_not_running_elsewhere` (one service of a name at a time: a one-off of `other/postgres`
  joins project `postgres` too). No host port check: compose run publishes no ports.
- **Dependencies (Decided here).** `run` starts what the definition declares as required, as
  `up` does, and nothing more:
  - `x-pcm.depends_on`: started if not running, waited for, provisioned (`start_deps`), so the
    one-off gets `PCM_<DEP>_*` exactly as the service does on `up`. Without this a one-off of an
    app needing postgres has no `PCM_POSTGRES_URL`.
  - **Not** the implicit ones of an ingress service (the active proxy, its hosts' edges):
    a one-off gets no route, so they serve it nothing. `edges_publish` is not run either.
    `start_deps` takes a third argument, `declared`, to read only `x-pcm.depends_on` for the
    service itself; its dependencies are started in full, as `up` starts them.
  - Compose's own `depends_on` of the chosen compose service (siblings in the same definition):
    the ones not running are started with `up -d` through the ordinary override (the same
    containers `pcm up` makes, with their labels and ingress), then waited for. The one-off then
    runs `--no-deps`, so compose never starts siblings from the one-off override (they would
    lack their ingress labels and carry no oneoff mark). The service's other containers are not
    started.
  - Nothing it started is stopped when it exits: they are ordinary running services.

### 3. The one-off override

`compose_flags` gains a mode, `PCM_ONEOFF=true` (set by `run` only): the override is
`$PCM_CACHE_HOME/<name>.run.override.yml` (the service's own is untouched, so a concurrent `up`
or `down` reads its own), and in it:
- the labels gain `io.pcm.oneoff: "true"` (`PCM_LABEL_ONEOFF`);
- no ingress labels (the proxy would route to a one-off as a second backend) and no network
  alias (a role provider's alias would resolve to the one-off too);
- everything else as `up`'s: id/name/source labels, the shared network, mount sets, toolkit,
  `memory:`, the snapshot image. compose builds or pulls the image as it does for `up`.

### 4. Telling it apart (`label!=io.pcm.oneoff`; podman 6 ANDs it with `label=`, checked)

- Left out: `project_containers` (so `wait_healthy`, `start_deps`' "is it running",
  `shutdown_order`, `check_ports_free`, `check_not_running_elsewhere`, snapshot, reset),
  `_box_container` (`exec`, `shell`), snapshot's commit lookup, `pcm_containers` (`pcm list`'s
  STATUS, `pcm show`'s status, the memory check's running set).
- Shown: `pcm ps` (all of pcm's containers, as podman lists them; the name says `-run-`), and
  `pcm show` gets a `one-offs` line naming the kept ones.
- Deleted: `pcm remove` removes them first (`podman rm -f` by `io.pcm.name` and
  `io.pcm.oneoff`; listed in the summary as `one-offs`) whatever the compose provider's `down`
  does with them; `implode` already deletes every container with `io.pcm.name`. The run
  override goes with `remove`, as the service's own does.

### 5. `pcm show` on podman 6

`inspect.sh`'s mounts format uses `{{$.Name}}` inside `range .Mounts`; podman 6 makes `$` the
list of inspect results. The format becomes `{{$n := .Name}}{{range .Mounts}}{{$n}}…`; a test
pins the exact string. The roadmap's Last item is ticked "in pcm/01-run".

### 6. Help, README, skill, AGENTS.md, version

- `commands.sh`: `run`, after `exec`; completion is the registry's (`service`, then words).
- `bin/pcm`'s epilogue: a paragraph on one-offs, and that a stock image is `podman run --rm`.
- `packages/pcm/README.md`: a "One-off commands" section with the same.
- `skills/pcm-containers`: "Snapshots, exec and shell" becomes "Snapshots, run, exec and shell"
  with `run` and the `podman run --rm` sentence.
- Core `AGENTS.md` "pcm: containers": one bullet (the oneoff label and what leaves it out).
- `packages/pcm/package.yml`: 0.2.0 (a new command; set by hand, which the hook respects).

## Test Spec (`packages/pcm/tests/run.bats`; podman and the compose runner stubbed)

- usage: no service; an unknown service; an unknown option before the service.
- the compose line: `run --rm --name <name>-run-<hex> --no-deps [-T] <svc> <cmd...>`; `--keep`
  drops `--rm` and names the container; `-u` passed; `--` and an undeclared option (`ls -la`)
  kept in the command; no command adds none; `-T` without a terminal.
- `-s`: the only compose service by default; several without `-s` is an error naming them; an
  unknown `-s` is an error.
- the one-off override: `io.pcm.oneoff: "true"` and the id/name/source labels; no
  `io.pcm.ingress` label for an ingress service; the service's own override not written.
- dependencies: a declared `x-pcm.depends_on` goes through `start_deps` (stubbed), the implicit
  proxy of an ingress service does not; a compose sibling not running is started with `up -d`
  before the one-off, a running one is not.
- the exit status is the command's.
- `project_containers`, `_box_container`, `pcm_containers` pass `label!=io.pcm.oneoff`.
- `remove` removes the one-offs (`podman rm -f`), and lists them in its summary.
- `pcm show`: the exact mounts format string (the podman 6 fix).

## Verification

Every command through the job: `anfs work run pcm-run core -- <cmd> </dev/null`. No real
container, machine or image is created, started or removed; podman is stubbed in every test.

- [ ] Before the change, on the base commit: `ANFS_TESTS_PMM_PCM=1 bats packages/pcm/tests`
      (the person asked for pcm's files to be run for this job; the suite has not run since
      config/04), so failures that predate it are told from new ones.
- [ ] While working: `bash -n` on each file touched; `ANFS_TESTS_PMM_PCM=1 bats
      packages/pcm/tests/run.bats`.
- [ ] After: `ANFS_TESTS_PMM_PCM=1 bats packages/pcm/tests`, compared with before.
- [ ] Once, at the end: `packages/anfs/tests/all`.
- For the person after merge (in `log.md`): `ppm install -f core/pcm`, then a live `pcm run`
  against a real service on this Mac.
