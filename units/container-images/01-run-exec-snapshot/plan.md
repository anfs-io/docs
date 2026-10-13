---
---

# Plan 01 — `pcm run`, and snapshots as a group

## Context — read these files first

- `units/container-images.md` — the command table this plan makes true, and its decisions
- `units/machine-images/01-fold-pim-into-pmm/plan.md` steps 1a and 1b, and its `log.md` — how
  `pmm run` and `pmm snapshot` came out. They are the reference
- `packages/pmm/home/.local/lib/pmm/commands.sh` — the declarations to match word for word
- `packages/pmm/home/.local/lib/pmm/machine.sh` (`cmd_run`, `mach_gone_if_rm`) and `snapshot.sh`
  (`cmd_snapshot_list|create|remove`), `completion.sh` (`complete_snapshot`, `CLI_COMPLETE_ARGS`)
- `packages/pcm/home/.local/lib/pcm/commands.sh` — pcm's declarations
- `packages/pcm/home/.local/lib/pcm/box.sh` — `cmd_exec`, `cmd_shell`, `cmd_snapshot`, `cmd_reset`,
  `snapshot_forget`
- `packages/pcm/home/.local/lib/pcm/sources.sh` — `canon`, `svc_dir`, `all_ids`, `cmd_list`
- `packages/pcm/home/.local/lib/pcm/runtime.sh` — the labels, `compose_flags`, `override_file`
- `packages/pcm/home/.local/lib/pcm/compose.sh` — `compose_command`, `cmd_up`, `cmd_down`
- `packages/pcm/home/.local/lib/pcm/{remove,implode,inspect,validate,completion}.sh`
- `packages/pcm/tests/helper.bash` — `pcm_setup`, the podman stub, `define`
- `packages/dev/home/.local/lib/ppm/container.sh` and `packages/anfs/tests/roundtrip` — the two
  callers of `pcm snapshot`
- `skills/pcm-containers/SKILL.md` — "Snapshots, exec and shell"

## Overview

Two places where pcm and pmm differ for no reason, closed:

| | Today in pcm | After |
| --- | --- | --- |
| A container from an image, with no definition | `podman run`, which pcm then knows nothing about | `pcm run <image>` |
| Snapshots | `pcm snapshot <service> [name]`: saves with a name, lists without one, cannot delete | `pcm snapshot list\|create\|remove` |

`exec` and `shell` exist in pcm already and need one word changed.

**A run container is a service whose compose file pcm wrote.** `pcm run` generates a one-service
`compose.yml` under pcm's data directory and starts it the way `up` starts any service. That is
the whole design: every other command (`up`, `down`, `ps`, `list`, `show`, `exec`, `shell`,
`snapshot`, `reset`, `remove`, `implode`) already works on a directory with a `compose.yml`, so
none of them learns a second kind of thing. It is the same move as pmm keeping a run machine's
settings in `state.yml` beside its disk, and nothing is written under `~/.config`.

No backwards compatibility: the old `pcm snapshot <service> [name]` form is removed with no alias.

## Implementation

### 1. Where a run container is recorded

```
$PCM_DATA_HOME/run/<name>/compose.yml     the record: what `run` was asked for, as a compose file
$PCM_VOLUMES_HOME/<name>/                 its data, as for any service
```

- `PCM_RUN_HOME` (`$PCM_DATA_HOME/run`) is set in `bin/pcm` beside `PCM_VOLUMES_HOME`, and in
  `pcm_setup` in the test helper.
- Its id is `-/<name>`: `PCM_RUN_SOURCE=-` in `sources.sh`, as pmm's `PMM_RUN_SOURCE` is. The
  containers carry the usual labels, so `io.pcm.id` is `-/<name>`, `io.pcm.name` is `<name>` and
  `io.pcm.source` is `-`.
- In `sources.sh`, three functions learn about it and nothing else does:
  - `canon`: the catalogue first. When it has no such name and `$PCM_RUN_HOME/<name>/compose.yml`
    exists, the answer is `-/<name>`. `-/<name>` spelled out resolves the same way.
  - `svc_dir`: `$PCM_RUN_HOME/<name>` for an id in `-/`.
  - `all_ids`: the catalogue's, then one `-/<name>` per record. `effective_ids` is unchanged, so a
    bare `pcm validate` checks definitions only.
- A definition always wins the name. `run` refuses a name that is defined (step 2), so the two
  only meet when a definition is added later; the run container is then reached as `-/<name>`,
  and `check_not_running_elsewhere` already refuses to start one while the other runs.

### 2. `pcm run`

In a new `lib/pcm/run.sh`. The declaration, in `commands.sh` before `up`, in pmm's order:

```bash
cli_cmd run "A container from an image, started. No definition is written: its
settings are kept with pcm's data. The name is the image's, then <image>-2,
<image>-3. A command after -- replaces the image's own" --passthrough \
  --opt "--name NAME: Its name" \
  --opt "--rm: Remove it when it is stopped" \
  --opt "--no-start: Record it and start nothing" \
  --opt "--share PATH: Mount a directory: PATH:TARGET[:ro]" \
  --opt "-p --publish PORT: Publish a port: host:container" \
  --opt "-e --env VAR: Set a variable: NAME=value" \
  --opt "--user USER: Who it runs as" \
  --opt "--attach: Join the shared network" \
  --arg "image" --arg "[cmd...]"
```

`cmd_run`, following pmm's step for step:

1. **The name.** `--name`, else the image's last path component without tag or digest
   (`docker.io/library/debian:trixie` is `debian`), then `debian-2`, `debian-3` while the name
   is defined or recorded. The name must be a service name (the rule `validate` applies).
2. **Refusals**, exit 73 as in pmm: `<name> is defined (<id>): pcm up <name>` and `<name> already
   exists (pcm show <name>)`. `--rm` with `--no-start` is a usage error (64).
3. **The record.** `compose.yml` written with yq, one compose service named `app`:

   ```yaml
   # written by `pcm run`; copy to containers/<name>/compose.yml to make it a definition
   services:
     app:
       image: docker.io/library/debian:trixie
       command: [sleep, infinity]          # the words after --
       user: "1000"                        # --user
       ports: ["8080:80"]                  # -p
       environment: {NAME: value}          # -e
       volumes: ["/Users/me/src:/src:ro"]  # --share
       stdin_open: true
       tty: true
   x-pcm:
     rm: true                              # --rm
     attach: true                          # --attach
     allow_mounts: [/Users/me/src]         # one per --share, so validate passes
   ```

   Paths are written absolute and no variable is used, so the record needs no `.env.schema`.
   `stdin_open` and `tty` keep an image whose command is a shell (debian, alpine) running, as
   `podman run -dit` does (Open Questions, 2).
4. **Validate** the record (`validate_services "-/<name>"`). When it fails the record is deleted
   and the command stops: `<name> was not made: its settings do not validate`.
5. **Start**: `cli_call up "-/<name>"`, unless `--no-start`. A start that fails leaves the record
   (as a machine whose `up` fails stays made); `pcm remove <name>` clears it.
6. `anfs_result name <name> image <image>`.

`--attach` is read in `compose_flags` (`runtime.sh`): `x-pcm.attach: true` in the service's
compose file sets `attached=true`, beside `PCM_ATTACHED_SERVICES`. It is not limited to run
containers, since a definition may want the same.

`run` does not pull stale sources (`bin/pcm` does that for `up` and `install` only): it reads no
definition.

### 3. What the other commands do with a run container

| Command | Change |
| --- | --- |
| `up`, `ps`, `exec`, `shell`, `reset`, `snapshot ...` | None: they resolve the name through `canon` and work on its directory |
| `down` | After stopping, a named service whose compose file has `x-pcm.rm: true` is removed (`run_gone_if_rm`, called from `cmd_down`; pmm's `mach_gone_if_rm`). It removes without asking, as `--rm` said to |
| `list` | A row per record. `source` is empty, so `-` on a terminal and `null` in `--json`, as `pmm list` shows a run machine |
| `show` | Unchanged, and for a run container it ends with the record's YAML under a `# containers/<name>/compose.yml` line, as `pmm show` ends with a machine's settings |
| `path` | Prints the record's directory |
| `remove` | `cmd_remove` also deletes `$PCM_RUN_HOME/<name>` for an id in `-/`. For a defined service nothing changes: the definition is kept |
| `implode` | None: it works from the labels and deletes `$PCM_DATA_HOME` |
| `install` | None: a source has no run containers |

`complete_service` (`completion.sh`) adds the directory names under `$PCM_RUN_HOME`.

### 4. Snapshots as a group

In `commands.sh`, replacing the one `snapshot` command, with pmm's words:

```bash
cli_group snapshot "Saved copies of a service: its containers and its data"
cli_cmd "snapshot list" "A service's snapshots: NAME ACTIVE" --alias ls --arg "service"
cli_cmd "snapshot create" "Save the service as it is: containers committed to images,
its data copied; with no name, named for the time" --arg "service" --arg "[snapshot]"
cli_cmd "snapshot remove" "Delete a snapshot" --alias rm --opt "-y --yes: Do not ask" \
  --arg "service" --arg "snapshot"
```

In `box.sh`, `cmd_snapshot` is split:

- `cmd_snapshot_list`: the rows through `anfs_table "NAME ACTIVE" "name active"`. `active` is
  `yes` for the snapshot the service runs from (the `active` file) and empty otherwise. The
  `* name` lines are gone, and with them the last list in pcm that was not a table.
- `cmd_snapshot_create`: today's saving half. With no name it is `date +%Y%m%d-%H%M%S`, as in
  pmm. Its last line names the way back: `Saved snapshot <s> of <id> (pcm reset <name> <s>)`.
- `cmd_snapshot_remove` (new): refuses a snapshot that does not exist (`pcm snapshot list
  <name>`), and one that is active (`<name> runs from <s>: pcm reset <name> first`), exit 1.
  Asks through `anfs_confirm`. Then deletes the images labelled exactly
  `io.pcm.snapshot=<name>/<s>`, the saved data (`remove_data_dir`), and the record file.
  `snapshot_forget` (every snapshot of a service, for `remove`) is rewritten as a loop over it,
  so there is one piece of code that deletes a snapshot.
- `complete_snapshot` in `completion.sh`: the file names in
  `$PCM_STATE_HOME/snapshots/<CLI_COMPLETE_ARGS[0]>/` except `active`, so `reset <service> <TAB>`
  and `snapshot remove <service> <TAB>` complete, as in pmm.
- `cmd_reset` is unchanged apart from its hint (`pcm snapshot list <name>`) and its hand-written
  usage line, which becomes `cli_usage_err`.

Where the snapshots are kept does not change (`$PCM_STATE_HOME/snapshots/<name>/`,
`$PCM_DATA_HOME/snapshots/<name>/`, `localhost/pcm-snapshot/<name>-<service>:<snapshot>`), so
`_override_snapshot_image` and its test are untouched.

### 5. `exec`, `shell`, `down`

- `exec` exists and matches pmm's: `pcm exec <service> [-u USER] -- <cmd>...`. Its argument kind
  is renamed from `word...` to `cmd...`, so the two usage lines read the same. `-s --service`
  stays (Differences that stay).
- `shell` is unchanged.
- `down` gains `--alias stop`, which pmm has and CLAUDE.md already claims for both.

### 6. Callers and documentation

- `packages/dev/home/.local/lib/ppm/container.sh`: `pcm snapshot "anfs-test-$1" ...` becomes
  `pcm snapshot create` or `pcm snapshot list`, and the listing at line 132 reads
  `pcm snapshot list --json` (Open Questions, 4, for the shape of `ppm container snapshot` itself).
- `packages/anfs/tests/roundtrip`: the three `pcm snapshot` checks use the new commands, and the
  completion check looks for `snapshot` as before. One check is added: `pcm run` a container,
  `pcm exec` in it, `pcm rm -y` it.
- `bin/pcm`'s help epilogue: a paragraph for run containers under "Sources" (`run` records under
  `$PCM_DATA_HOME/run/<name>/`; listed with no source).
- CLAUDE.md: the verbs paragraph names the shared set once (`run`, `up`, `down`, `ps`, `list`,
  `shell`, `exec`, `snapshot list|create|remove`, `reset`, `remove`); the key table gets
  `pcm snapshot list` (`name`, `active`) and the note that `pcm list`'s `source` is null for a run
  container; the `pcm` rows under "Testing".
- `skills/pcm-containers/SKILL.md`: "Snapshots, exec and shell" uses the group, and gains two
  lines on `pcm run` and on copying `pcm show`'s YAML into `containers/<name>/` to keep one.
- `units/container-images.md`: the plan list marks this one written.

### Differences that stay

- **No `--profile`, `--cpus`, `--memory`, `--disk`, `--media`, `--arch`.** A container has no
  disk or boot media, and limits are not something pcm sets for any service today.
- **`--env`, `--attach` and a command after `--` are pcm's only.** A machine has no equivalent.
- **`--no-start` creates nothing but the record.** A machine has a disk to make; a container is
  created by the `up` that starts it. `pcm list` shows the name with an empty status until then.
- **`--share` takes `PATH:TARGET[:ro]`**, where pmm's takes `PATH[:tag][:ro]`: a container mounts
  at a path, a machine by a tag.
- **`snapshot list` has `active`, pmm's has `build`.** A service runs *from* a snapshot until it
  is reset to its definition; a machine's disk is replaced by the snapshot's and nothing is
  active. `snapshot create` does not stop the service first; pmm stops the machine.
- **`exec` and `shell` keep `-s --service`**: a pcm service may be several containers.
- **pcm has no `ssh` alias for `shell`.**
- **`ps` goes on to `podman ps`**, with podman's columns and options.

## Test Spec

A new `packages/pcm/tests/run.bats`, and `box.bats` for the snapshot group. podman is the
helper's stub, extended per test to record its argv, as `machine.bats` does.

**run**

- The record: `run docker.io/library/debian:trixie --no-start` writes
  `$PCM_RUN_HOME/debian/compose.yml` with the image, and nothing under `$PCM_CONFIG_HOME`.
- Names: a second run is `debian-2`; `--name web` is `web`; a registry port and a digest are
  stripped (`localhost:5000/x/app@sha256:...` is `app`).
- Each option lands in the record: `-p` twice, `-e`, `--user`, `--share` (absolute, with
  `allow_mounts`), `--attach`, `--rm`, the words after `--` as `command`.
- Refusals: a defined name and a recorded name exit 73; `--rm --no-start` and no image exit 64; a
  bad name exits 1 and leaves no record.
- A record that does not validate (a `--share` path that does not exist) is deleted.
- `canon debian` is `-/debian`; a definition of the same name wins, and `-/debian` still resolves.
- `list --json`: the row has `"source": null`; piped, the cell is `-`.
- `compose_flags -/debian` writes the three labels with `-` as the source, and joins the shared
  network only with `x-pcm.attach`.
- `down` of a record with `x-pcm.rm` removes its record and data; without it both stay.
- `remove -y debian` deletes `$PCM_RUN_HOME/debian`; `remove` of a defined service leaves its
  definition.
- `pcm __complete up ""` offers a recorded name.

**snapshots**

- `snapshot list`: rows from the state files, `active` on the right one; `--json` keys are `name`
  and `active`; a service with none prints nothing and exits 0.
- `snapshot create` with no name uses the time (matches `^[0-9]{8}-[0-9]{6}$`).
- `snapshot remove`: an unknown one and the active one are refused; without `-y` and no terminal
  nothing is deleted; with `-y` the record, the data and the images (the stub's `rmi` argv) go.
- `remove` of a service still deletes every snapshot (through the same function).
- `pcm snapshot db s1`, the old form, exits 64.
- Completion: `snapshot remove db <TAB>` lists `db`'s snapshots and not `active`.

**declarations**

- `pcm help` lists `run` before `up`; `pcm stop x` is `down`; `exec`'s usage reads `<service> <cmd>...`.

## Verification

- [ ] `bats packages/pcm/tests </dev/null`, and `packages/ppm/tests`, `packages/anfs/tests`
- [ ] On this Mac, with real podman and `PCM_*_HOME` pointed at a temp dir:
      `pcm run docker.io/library/debian:trixie`, `pcm list`, `pcm exec debian -- cat /etc/os-release`,
      `pcm shell debian`, `pcm snapshot create debian s1`, `pcm snapshot list debian`,
      `pcm reset -y debian s1`, `pcm snapshot remove debian s1` (refused: active),
      `pcm reset -y debian`, `pcm snapshot remove -y debian s1`, `pcm rm -y debian`
- [ ] `pcm run --rm --name once docker.io/library/alpine:3`, then `pcm down once`: no container,
      no record, no data directory
- [ ] `pcm run -p 8080:80 docker.io/library/nginx:alpine`, and `curl localhost:8080` answers
- [ ] After all of it, `podman ps -a --filter label=io.pcm.id` and `find` under the temp homes
      show nothing left
- [ ] `ppm container snapshot debian` and `ppm container list` still work against a test box
- [ ] `packages/anfs/tests/roundtrip debian --nested` passes (pcm's containers really start there)

## Open Questions

Each has a recommendation; the plan above is written to it.

1. **`--share PATH:TARGET[:ro]`, or podman's `-v`?** `--share` is pmm's word and the unit's goal
   is one API. Recommended: `--share`, with the target required (there is no sensible default
   path inside an unknown image).
2. **Keeping a shell image alive.** `pcm run debian` with no command would exit at once, because
   the image's command is `bash` with no terminal. Recommended: the record sets `stdin_open` and
   `tty`, as `podman run -dit` would, so it stays up and `pcm shell` works. The cost: `podman
   logs` of such a container shows terminal control characters.
3. **`snapshot list`'s second column.** pmm's is `build`, which pcm has nothing for. Recommended:
   `active`, the one thing a person needs before `reset` or `snapshot remove`.
4. **Should `ppm container snapshot <distro> [name]` become a group too?** It is the same
   two-meanings command. Recommended: yes, in this plan (`ppm container snapshot list|create`),
   since its two lines are being edited anyway and the rule is "the verbs are the same
   everywhere". Leave it if `core/dev` is about to move to its own source.
5. **May a definition's `x-pcm.depends_on` name a run container?** With `canon` extended it
   would resolve. Recommended: no. A definition in git should not depend on something made by
   hand on one machine: `service_deps` resolves through the catalogue only, and `validate` reports
   the name as unknown.
6. **`--cpus` and `--memory`.** They mean something for a container (limits), and leaving them
   out is the one place a pmm habit fails in pcm. Recommended: leave them out now; add them to
   `run` and to `validate`'s vocabulary together if limits are ever wanted for defined services.
