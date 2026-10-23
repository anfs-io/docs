---
plan: agents/03-ui
job: agents-03
date: 2026-10-23
---

# Log: agents/03, anfs-ui on the bus

## What was built

- **The bus client** (`containers/ui/lib/anfs/bus.rb`, `bus_config.rb`, `bus_status.rb`). It
  uses nats-pure 2.7.0 (2026-10-07) and connects as `reader`. On first start it makes the durable
  consumer `ui` on stream `ANFS` (filter `anfs.>`, explicit ack, deliver `new`). It pulls up to
  100 messages per pull, each pull open for 30 s, and handles every message as it comes, then acks
  it. The client's own reconnect is off: the bus loop retries after 1, 2, 5, 10, 30, then every
  60 s, and starts again at 1 s after a connection that worked. Errors are named by step: `cannot
  connect to <url>`, `cannot read stream ANFS (is anfsd running?)`, `lost the connection`,
  `cannot read the bus password (<file>): ENOENT`. The log gets one line per change of state. The
  password, and any user or password in the URL, are redacted from every log line and state.
  An event the handler fails on is logged and acked, and is not taken as a connection failure.
- **What an event updates** (`poller.rb`, `container_rows.rb`):
  - `container.*` events change the service's row in Containers from the event's data, with no
    command. The row is the one for `data.pcm` that is not shadowed. It reads `running` while one
    of the service's known containers runs, `stopped` when some exist, and not created when none
    is left (`removed`). Anything that arrives while a `pcm list` runs is applied again over the
    list's result. `Store#update` makes each change in one step, so a list that lands at the same
    time is never overwritten with older rows.
  - `podman.connected` and `podman.disconnected` cause one `pcm list`.
  - `source.updated` causes one `anfs src list`.
  - `report.*`, `run.*`, `pod.*` and `container.health` go to the events page and nothing else.
  - No event re-lists another kind, and none re-lists everything. Reads go through the old
    per-kind 2 s guard. Re-renders go through `Coalescer`: the first runs at once, then at most
    one per key every 2 s.
- **The reconcile:** every kind is listed at start, every 5 minutes (`Anfs::RECONCILE`), and when
  a command run from its row ends. The kinds' reconciles are spread across the 5 minutes, so the
  API gets one list about every 33 s instead of nine at once. The old intervals (5 s for
  containers and machines, 30 to 300 s for the others) are gone.
- **The watch is gone.** In the ui: `Anfs::Watch`, `LogFollower`, `Poller#event`/`#log_line`
  and `RPC#watch`. In `anfs rpc`: `_rpc_watch` and `_rpc_events`. `{"watch":true}` now gets
  `{"error":"unknown request (run or version)"}`. The overview used to re-render on every hub key,
  including each line of a job's output. It now follows only the kinds and `bus`.
- **`/events`** (in the nav): the latest 100 events, newest first, with time, type, subject,
  space and by. The space is `spacepath`, or the scope id, or `host`; by is `actor`, plus `as
  <role>` when there is one. The bus's state is at the top. The overview has the same state line,
  linked to the page.
- **Credentials:** `compose.yml` mounts `${HOME}/.local/state/nats/users/reader` read-only at
  `/run/secrets/anfs-bus-reader`, allowed by `x-pcm.allow_mounts`. `ANFS_BUS_PASSWORD_FILE` points
  the ui at it. The address is `ANFS_UI_BUS_URL` in `.env.schema`, default
  `nats://host.containers.internal:47222`, which `local.env` can override.
- Docs: core AGENTS.md (the API's table and its grant, the ui's layout entry, "Events", "The
  bus"), the ui's README (the bus, the events table, the settings, rebuilding the image),
  `packages/nats/README.md`. The ui is at 0.2.0.

## Commits (branch agents-03, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| docs | `47c2e6d` | plan.md |
| core | `6037d84` | the ui on the bus, the watch removed from `anfs rpc`, docs and tests; ui 0.2.0 (anfs stays 0.2.17: the hook says it was already bumped since its last push) |
| docs | (this) | this log |

## Which kinds still poll, and why

Every kind still gets the 5-minute reconcile, which also covers a missed event and a bus that
is down. These kinds get nothing else, because no event for them exists in 01 or 02:

| Kind | Tool | Why there is no event |
| --- | --- | --- |
| Stacks, Packages | anfs, ppm | ppm and `anfs install` emit no `package.*` or `stack.*` |
| Images, Machines | pmm | pmm emits no `machine.*` or `image.*` (and no VM event source exists like `podman events`) |
| Spaces, Repositories | wsm | `wsm install` and `wsm remove` emit none (01 gave wsm ids and `events ls` only) |
| Skills | psm | psm emits none |

A change made from the ui's own rows still shows at once, because the job's end re-reads its
kinds. A change made in a terminal (`ppm install`, `pmm up`) shows at the next reconcile, up to
5 minutes later. Before this plan the watch showed it within seconds, at the cost of the loop
risk. The fix is emitters, in a later plan: `package.installed` and `.removed` (ppm),
`machine.started`, `.stopped` and `image.built` (pmm), `space.installed` and `.removed` (wsm),
`skill.synced` (psm). Each would get one line in the poller's `READS`, or a data-driven row like
containers.

Sources update on `source.updated` with one list of the kind (`anfs src` has no `show`).
Packages that turn `outdated` after an `anfs src update` wait for the reconcile.

## Decisions made during the work

1. **The password is a mounted read-only file, not an environment variable.** The toolkit's
   token (`ANFS_RPC_TOKEN`) is in the environment, but an environment shows in `podman
   inspect`. A mount puts nothing in the image, the environment, compose's override or any log,
   and a rotated password needs no rebuild. The ui reads the file at every connect.
2. **No `bus: read` key in pcm** (plan 02's idea). For one container it would do what two lines
   of `compose.yml` do. It can come when a second container needs the bus. The cost: the mount
   names `~/.local/state/nats` (not `$XDG_STATE_HOME`), and a moved port goes in `local.env` as
   `ANFS_UI_BUS_URL`. pcm's own `validate_services` (a throwaway bats in the job, deleted) passes
   the definition and refuses the mount without `allow_mounts`.
3. **The watch is removed from `anfs rpc` too**, not only from the ui. The ui was its only client,
   and the brief said it goes with no backward compatibility.
4. **nats-pure's `fetch` holds a batch until it is full or the pull expires** unless it is given
   a block. Without the block, events published right after a reconnect waited the whole 30 s
   (found against a throwaway server). The bus uses the block form, and a spec covers it.
5. **The client's reconnect is off** (`reconnect: false`), so there is one retry loop with one
   backoff and one state. A server shutdown ends the pull at once (status 409), so the drop is
   seen immediately, not after a ping timeout.
6. **Container rows come from the events since the last list.** A service with several containers
   (authentik) reads `stopped` if one of them stops on its own while another runs, until the next
   list (up to 5 minutes). `pcm down` and `pcm up` move all of them and read right. The other
   ways (re-listing on every event, or `pcm ps`/`pcm show`, which are text and heavier) were what
   the brief ruled out. `container.health` changes no row: pcm's list shows no health.
7. **Events from the durable consumer's backlog are applied in order** after the first lists. The
   last event per container wins, so the result converges. Nothing is skipped by time: the
   container's clock and the host's can differ.
8. **The reconciles are spread** (`start`): every kind is listed at once, then kind *i* first
   reconciles after (*i*+1)/9 of the interval. This keeps the acceptance test's rate check well
   under 10 a minute except in the minute of a restart.
9. `Poller#nudge("anfs")` no longer wakes every kind, only the kinds `anfs` lists. No kind with a
   row action has `anfs` as its tool, so nothing that used it is lost.
10. Gems for the host's rspec went into the session's scratch bundle path (`BUNDLE_PATH`), so the
    host's mise Ruby got nothing new. Gemfile and Gemfile.lock hold nats-pure 2.7.0 and its
    dependencies (base64, concurrent-ruby, securerandom, uri), with checksums.

## Tests

- **rspec** (`bundle exec rspec` in `containers/ui`, on the host, nats-server v2.15.1 on PATH):
  **90 examples, 0 failures**, the same on three runs. rubocop: 36 files, no offenses.
  - `spec/anfs/bus_spec.rb` (new). With a fake client:
    - the consumer is made once with the expected config, and an existing one is kept;
    - each event is yielded and acked; non-JSON and non-objects are acked and skipped;
    - a handler's failure is logged and is not the connection's;
    - the backoff runs 1, 2, 5, 10, 30, 60, 60, 60 with one log line, and starts again after a
      connection;
    - a missing password file is named;
    - the password, and a URL's user and password, are in no log line or state;
    - `Config.from_env` on the host (the port file, users/reader) and in the container.

    Against a throwaway nats-server (random port, temp store, the reader's exact permissions; a
    spec checks that they match `lib/anfs/bus.sh`):
    - an event arrives as it is published and is acked (nothing pending);
    - a restarted ui resumes where it left off and never sees history from before its consumer;
    - with the server stopped: unreachable, **under 0.1 s of CPU in 2 s**, then connected again
      with one log line, and the next event arrives;
    - the reader is refused a publish to `anfs.host.x.y`.

    The server cases skip with a message when no nats-server is on PATH, as in the container.
  - `spec/anfs/poller_spec.rb` (rewritten). Container events, with a strict CLI double that
    fails on any command:
    - died, removed, created, started give stopped, not created, stopped, running, on the winning
      row only, with the other fields kept;
    - a two-container service stays running while one runs;
    - health, pods, a container that is not pcm's and an unknown service change nothing;
    - an event during a list is applied again over the list.

    Reads and re-renders:
    - one event of each kind gives one extra `anfs src list` and one extra `pcm list`, and
      report, run, pod and container events give no list;
    - a burst of 400 gives one read per kind;
    - events every 0.1 s give reads at least one window apart;
    - 400 events give 2 re-renders of containers and 2 of the events page;
    - each kind reconciles on its interval, the reconciles are spread, and a nudge re-reads only
      its tool's kinds.

    Also: the events page's fields; **an idle poller uses under 0.04 s of CPU in 2 s**; the
    Coalescer.
  - `app_spec.rb`: the events page (fields, newest first, its stream), the unreachable state with
    its reason and "every 5 minutes", the overview's bus line. `event_stream_spec.rb`: the overview
    follows the kinds and `bus`, not `events` or a job's output. `store_spec.rb`: `update`.
- **A smoke run** (not a test file): puma on the host with `RACK_ENV=production`, against a
  throwaway nats-server, with `ANFS_BIN` an empty directory (no real tool run). Two events
  published as admin appeared on `/events` with the right fields, and the overview said `Bus
  connected`. **Puma's CPU time did not move over 10 s idle** (0:00.29 both times), and the
  password is not in its log.
- **bats**: `rpc.bats` 22/22 alone (the two watch tests replaced by one: `{"watch":true}` is an
  unknown request and the next request is still answered).
- **The full run, once** (`anfs work run agents-03 core -- packages/anfs/tests/all </dev/null`):
  **739 passed, 1 failed**, 54 files (pmm and pcm off), 85 s. The failure was `rpc.bats` "a
  group's list is a read" at its second assertion (`anfs space skill ls`). It passed in 4 runs
  alone and in 8 runs in parallel, printing the right answer each time. That code (anfs to wsm
  routing) is not what this plan touched. It was not run again, per the brief's "once".
- The real bus was only read: a `nats stream info ANFS` as `reader`, with the password in the
  environment, not argv (17 messages, 2 consumers, no `ui` yet). Nothing was published, and no
  consumer was made on it.

## Not run

- No container was started, and the real ui and bus were not restarted. So these are unverified:
  the mount of the 0600 password file (in a 0700 directory) being readable in the podman VM, and
  `host.containers.internal:47222` reaching the host's loopback NATS. The toolkit API reaches the
  host's loopback 47100 the same way, so the address should work. The by-hand test below checks
  both: the page says `Bus connected`, or names the failing step.
- Linux: how a rootless container reaches the host's `127.0.0.1:47222` (open, below).
- pmm's and pcm's suites (off); CI (nothing pushed).

## By hand, for the person after merge (the acceptance test)

1. **Merge and update the source** (core is linked from its source, so this also brings `anfs
   rpc` without the watch):

   ```bash
   # after the ff-merge of agents-03 into core main
   anfs src update core
   anfs bus status                    # nats, jetstream, anfsd, podman: ok
   ```

2. **Rebuild the ui's image and restart it.** There is a new gem, and `pcm up` reuses the image
   it has:

   ```bash
   pcm down ui
   podman rmi localhost/anfs-ui:dev
   pcm up ui                          # builds the dev image (bundle install), then starts it
   pcm logs ui | grep bus             # "anfs-ui: bus connected: nats://host.containers.internal:47222"
   ```

   If it says `bus unreachable: ...` instead, the reason names the step. `cannot read the bus
   password (/run/secrets/anfs-bus-reader)` means the mount: check `ls -l
   ~/.local/state/nats/users/reader`. `cannot connect to ...` means the address. Report it with
   `anfs report`.

3. **Open the ui**: http://localhost:9292. The overview's first line says `Bus connected since
   ...`. Click it, or **Events** in the nav, to get `/events`. It is empty at first: the consumer
   `ui` was just made and takes new events only. `anfs bus nats -- consumer info ANFS ui` shows
   it.

4. **Watch the events page while valkey goes down and up** (valkey is safe: nothing running
   depends on it). In a terminal:

   ```bash
   pcm down valkey
   pcm up valkey
   ```

   On `/events`, without reloading, newest first: `container.died` (subject `valkey_valkey_1`,
   space `host`, by `podman`), `container.removed`, `pod.removed`, then `pod.created`,
   `container.created`, `container.started` and `container.health`. On `/containers` the valkey
   row goes to not created, then stopped for a moment (`created`), then running, as the events
   come, with no `pcm list` behind it.

5. **File and close a scratch report.** Use a space with no `id:` (`user/evals`), so the report
   goes to the host scope. anfsd's report sink then leaves `docs/BACKLOG.md` alone (it renders
   only reports in anfs-dev's space), and anfs-dev's dispatch rules (`report.filed` in anfs-dev)
   cannot fire:

   ```bash
   export ANFS_REPORT_TO=~/spaces/user/evals/bus-check.md
   anfs report --title "ui bus check" --actual "testing the ui's events page"   # prints r-<date>-<n>
   anfs report close r-<date>-<n> --commit $(git -C ~/.local/share/anfs/sources/core rev-parse --short HEAD)
   rm ~/spaces/user/evals/bus-check.md
   unset ANFS_REPORT_TO
   ```

   `/events` shows `report.filed`, then `report.closed`, subject `r-<date>-<n>`, space `host`, by
   your actor. The record stays in `~/.local/state/anfs/reports/host/` and the events stay in the
   files and the stream, like every event.

6. **The API's request rate and the CPU.** Give it two minutes after the restart (the restart's
   own minute has the nine first lists), then:

   ```bash
   grep -c "$(date +%Y-%m-%dT%H:%M).*rpc --token" ~/.local/state/anfs/anfs.log   # about 2 (each kind every 5 min, spread); was ~440 before 06
   top -l 1 | grep 'CPU usage'                                                    # idle near 95%
   podman stats --no-stream ui_ui_1                                               # CPU % near 0
   ```

   The bound is under about 10 a minute. A `pcm down`/`up` adds no lines, because container
   events need no list. A `podman machine stop`/`start` adds one `pcm list` each.

7. **Optional, the bus down** (it stops the event log for everything, so it is your call):
   `anfs bus stop nats`. Within a second the overview says `Bus unreachable since ...: lost the
   connection: ...`, then `cannot connect to ...; next try in 2s`, ... up to 60 s. `top` stays
   idle. `anfs bus start` brings `Bus connected` back within a minute. Any events in between
   arrive afterwards, because the consumer is durable.

## Open questions

1. **Emitters for the kinds that only reconcile** (the table above): a plan of their own (ppm,
   pmm, wsm, psm), or one with the next change to each tool.
2. **Linux:** a rootless container reaching the host's `127.0.0.1:47222`. pasta's `-T 47222` or
   `--map-host-loopback`, or NATS also listening on a unix socket (it cannot), or a websocket
   listener. To settle on a Debian host, with `bus: read` in pcm if it needs a per-OS override
   like `toolkit:`.
3. **Multi-container services** (decision 6): if a partial stop matters before 5 minutes, anfsd
   could add the io.pcm.id label to the data, and pcm a cheap `pcm list <name> --json`.
4. **The `rpc.bats` flake** seen once in the full run. Watch for it, and look at it if it comes
   back.
