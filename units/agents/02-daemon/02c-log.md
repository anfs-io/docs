---
plan: agents/02c-podman
job: agents-02c
date: 2026-10-23
---

# Log: agents/02c, podman events into the bus

## What was built

- **anfs-io/runtime** (anfsd): `internal/podman`. One child, `podman events --format json
  --filter type=container --filter type=pod` (from now). Kept: a container's `create`, `start`,
  `stop`, `died`, `remove`, `pause`, `unpause`, and `health_status` only when that container's
  health changes; a pod's `create`, `start`, `stop`, `remove`. They are written through
  `cloudevent.Append` into the host scope as `container.<verb>` / `pod.<verb>` (`created`,
  `started`, `stopped`, `died`, `removed`, `paused`, `unpaused`, `health`), with actor `podman`,
  the name as subject, and data `id` (12), `image`, `pod`, `exit` (died), `health`, `pcm` (the
  `io.pcm.name` label). The rest is dropped. The guards: one event per container, status and
  second; at most 20 a second, with the rest counted and logged once a minute. Podman away: the
  child is started again after 5, 10, 30, then every 60 s, with one log line per change, plus
  `podman.connected` and `podman.disconnected`. `~/.local/state/anfsd/podman.json` holds the
  state for `anfs bus status`. Also `ANFSD_PODMAN=on|off` in `internal/config`, the wiring in
  `cmd/anfsd`, `testdata/events.jsonl` (recorded here), a fake podman in `internal/testfake`,
  README and AGENTS.md.
- **core**: `anfs bus status` gets a `podman` row; `anfsd.env` gets `ANFSD_PODMAN`;
  `events.sh` completes `container.*`, `pod.*` and `podman.*`. Docs: AGENTS.md ("Events": the host
  scope's container and pod events; "The bus": Podman, the row, `podman.json`), README, anfsd's
  README. anfsd is at 0.3.0 and `pins.yml` names v0.3.0.

## Commits (branch agents-02c, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| runtime | `ae71386` | `internal/podman`, its wiring, `ANFSD_PODMAN`, the fixture, the fake podman, tests, the flaky 02b test fixed, README, AGENTS.md |
| core | `5293c0c` | the `podman` row, `ANFSD_PODMAN`, the completion types, docs, tests; anfsd 0.3.0, pins v0.3.0 (anfs stays 0.2.17: the hook says it was already bumped since its last push) |
| docs | (this) | this log |

## Decisions made during the work (deviations from the plan, and why)

1. **The fixture is not from a `pcm up`/`down` in the job.** No container was started or
   stopped (the person's rule for this job). Instead it comes from a read-only `podman events
   --stream=false --since 24h --format json` on this Mac (podman 6.1.1, 33,041 lines). Two
   lifecycles were kept: a busybox container with exec noise that died 137, and pcm's glitchtip
   coming up and going down in its pod (health checks, exec, sync, cleanup). 44 lines in all;
   `/Users/roberto` is `/Users/person` in the labels.
2. **Podman reports no `stop` for `pcm down`.** In 24 h of recorded events no `stop` status
   appears at all: a compose down is `died` then `remove`. The kinds seen were attach, cleanup,
   create, died, exec, exec_died, health_status (29,375 of them), init, remove, start, sync, and
   pod create and remove. `container.stopped` stays mapped as the plan says, for when podman emits
   it. So the person's check (below) expects `container.died` and `container.removed`, not the
   plan's `container.stopped`.
3. **`pod` is the pod's id (12 characters), not its name.** A container's event carries only
   `PodID`. A name would need a lookup or a map built from pod events, which a pod created before
   anfsd started would not be in.
4. **When "connected" starts:** the child has been up 2 s, or has printed a line. On a stopped
   machine podman fails at once. **The backoff resets only after a connection lasted a minute**,
   so a podman that connects and drops over and over still climbs 5, 10, 30, 60 s and never
   restarts every 5 s. The plan said neither.
5. **`podman.json`** (`state` connected|away|off|starting, `since`, `error`, `next`, written by
   temp file and rename on each change and each try) is how `anfs bus status` learns the state:
   anfsd has no RPC. The row reads it only while `anfsd.pid` is alive. Otherwise it is `stopped:
   anfsd is not running`, and a missing file says an older anfsd. `away` makes the exit 1, like any
   non-`ok` row. `off` does not (`_bus_row` gained a "soft" argument).
6. **`podman.disconnected` carries `data.error`**: podman's last stderr line, or "podman events
   ended".
7. **The child runs in its own process group**: SIGTERM goes to the group when anfsd stops, with a
   5 s grace period. The drop counter's one-minute log is a timer armed only while there are drops
   (no ticker when idle).
8. **Test isolation**: the fake podman is in `testfake`, so every test that uses the fakes' PATH
   finds it first. It defaults to `away`, which is a stopped machine. 02a's and 02b's e2e tests
   run anfsd with `ANFSD_PODMAN=off`: their PATH is `/usr/bin:/bin`, and on Linux (CI) that holds
   a real `/usr/bin/podman`.
9. **02b's `TestGuardsBookkeeping` was flaky** (about one run in two here, on main as well): it
   ended while its fake run was still sleeping, and the run's wrapper wrote its exit status into a
   temp dir that was being removed. The test now waits for both runs to complete.
10. `podman.connected` and `podman.disconnected` are in the completion types too.

## Tests

- **Go** (`go test -race ./...` in `runtime/go`; three runs with `-count=3`, none failing; gofmt
  and vet clean): **76 tests** (02b: 67).
  - `internal/podman` (8 new):
    - The recorded lines give exactly the 12 expected events: types, subjects, data, all valid
      host-scope lines. The noise gives none, and health only on a change.
    - Dropped types write nothing.
    - 50 identical starts in one second are 1 event. A burst of 200 is cut at 20, and the drops
      are logged once.
    - The backoff steps.
    - The fake exits and comes back: connected, `container.started`, disconnected (with its
      error), connected; the backoff asked for tries 1 and 2.
    - Away: five tries, no event, one log line, a state with the error and the next try.
    - Not installed.
    - Off.
  - `cmd/anfsd`: the idle test now runs with podman away the whole time: **82 ms of CPU over 11
    s** (the limit is 200 ms; 02b's was 44 to 65 ms), podman tried at most 3 times, one `podman
    away` line. **`TestPodmanOff`** (new): `ANFSD_PODMAN=off` never starts podman, and
    `podman.json` says off.
  - `config`: `ANFSD_PODMAN` is parsed, and a bad value is refused.
- **e2e** (`go/test/e2e.bats`, with the pinned nats-server v2.15.1 and nats v0.5.0 from `~/.local/bin`,
  on a random port in a temp HOME): **5 passing** (+1). The fake podman prints the recorded
  glitchtip lifecycle. Then `anfs.host.container.started` holds 1 message (subject
  `glitchtip_web_1`, actor `podman`, `pcm=glitchtip`, `id=2f820df40bb2`), with created 1,
  health 2, died 1, removed 1, `podman.connected` 1, nothing for exec or sync, and `podman.json`
  says connected.
- **core** (`packages/anfs/tests/all`, once at the end): **741 passed, 0 failed**, 54 files (pmm
  and pcm off), 58 s (02b: 740). bus.bats gained one test for the podman row's states: no file,
  away (exit 1), off (exit 0), connected, anfsd gone. The existing status tests now include their
  podman row. anfsd.bats names `ANFSD_PODMAN`; events.bats checks that completion offers
  `container.*`.
- No fake podman or `sleep` was left running after the runs (checked with `pgrep`).

## Not run

- The real podman was run only as `podman version` and `podman events --stream=false --since 24h
  --format json` (read only), plus once `podman --connection nonexist events` (it fails before
  connecting). No container, pod or machine was started, stopped or removed.
- Nothing on this host was installed, loaded or restarted. The running bus (47222) and anfsd were
  not touched, apart from reading the PATH in `~/Library/LaunchAgents/io.anfs.anfsd.plist`: it
  includes `/opt/homebrew/bin`, so the service finds podman.
- A real podman machine stopping under a running `podman events` was not observed here (that
  would mean stopping the machine). The design assumes the child exits when the connection goes.
  Step 4 below checks it.
- pmm's and pcm's suites (off); CI and the release (nothing pushed); Linux.

## By hand, for the person after merge

1. **Rebuild anfsd from source and restart it** (the release v0.3.0 can follow: push runtime, `git
   tag v0.3.0 && git push origin v0.3.0`, then the sums into core's `packages/anfsd/pins.yml` with
   `packages/anfsd/tests/pins --print`; this session can do that for you):

   ```bash
   cd ~/spaces/anfs-dev/runtime/go          # main, with agents-02c merged
   go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
   anfs bus restart anfsd
   anfs bus logs anfsd                      # "podman connected: /opt/homebrew/bin/podman events"
   ```

2. **Reinstall** (the new `anfs bus status` row, `ANFSD_PODMAN` in anfsd.env, the completion
   types):

   ```bash
   anfs src update core
   ppm install core/anfsd                   # rewrites and reloads the unit (ANFSD_BIN)
   ```

3. **Status**: `anfs bus status`. A row `podman  ok  connected since <time>: container.* and pod.*
   into the host scope`. `anfs events ls --type 'podman.*'` shows one `podman.connected` from the
   restart.

4. **A container change appears.** Use **valkey** (stack/valkey). It is running, and nothing
   running depends on it: only glitchtip and twenty use valkey, and neither is up. Not ui (yours to
   look at), not postgres (authentik uses it), not authentik (a login), not telemetry (three
   containers, and others may send it traces). In one terminal:

   ```bash
   anfs events ls --type 'container.*' -f
   ```

   In another:

   ```bash
   pcm down valkey
   pcm up valkey
   ```

   Expected, all with subject `valkey_valkey_1` and BY `podman`: `container.died` (data
   `exit=0`), then `container.removed` from the down. Podman reports no `stop` for it (deviation
   2). From the up: `container.created`, `container.started`, then `container.health`
   (`health=starting`, then `healthy` once its check passes). `anfs events ls --type
   'container.*' --json | tail -n 1` shows the data (`pcm=valkey`). On the bus: `anfs bus nats --
   stream subjects ANFS 'anfs.host.container.>'`.

5. **Optional, the machine going away** (your call: it stops every container): `podman machine
   stop`. `anfs bus status` then says `podman  away  <podman's error>; next try ...` (exit 1),
   and `anfs events ls --type 'podman.*'` shows a `podman.disconnected`. After `podman machine
   start` (and `pcm up` of what you need) it reconnects within a minute: another
   `podman.connected`. `ps -o time= -p $(cat ~/.local/state/anfsd/anfsd.pid)` should barely move
   while it is away. If `podman events` instead hangs when the machine stops (step 4's assumption
   fails), the status stays `connected` and no `podman.disconnected` comes. Report it with `anfs
   report`.

To turn it off: `ANFSD_PODMAN=off` in `~/.config/anfsd/anfsd.local.env`, then `anfs bus restart
anfsd` (the row says `off`, which is not a failure).

## Open questions

1. `anfs bus status` exits 1 while the podman machine is stopped (`away`), which on macOS is a
   normal state. If a script reads `status` as health, `away` could become soft like `off`.
2. `pod` is an id. If 03's ui wants pod names, anfsd could map ids from the `pod.created` events it
   sees, or the ui can list once on `podman.connected`, which it does anyway.
3. Podman's `health_status` arrives every check (here about 30,000 a day). anfsd reads and drops
   them in a few microseconds each, but it is the one steady input with podman connected.
