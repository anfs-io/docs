---
plan: agents/02a-bus
job: agents-02a
date: 2026-10-23
---

# Log: agents/02a, the bus

## What was built

- **anfs-io/runtime** (first commit): `go/` is one module (`github.com/anfs-io/runtime/go`, `go
  1.27.3`). `cmd/anfsd` (`run`, `ship --once`, `version`, `help`); `internal/config` (anfsd.env
  and anfsd.local.env read with core's settings-file rules, the environment first; the XDG paths;
  NATS's `port` and the `anfsd` password from `~/.local/state/nats`); `internal/cloudevent` (the
  anfsv 1 profile: `Parse` validates a line, `Line`/`Append` write one: UUIDv7, one `O_APPEND`
  write, at most 4096 bytes, the bash writer's attribute order); `internal/spool` (month files,
  complete lines past a cursor per file, `cursors.json` by temp file, fsync, rename; `Watch`:
  fsnotify on `events/` and each scope, plus a 30 s reconcile); `internal/bus` (connect as `anfsd`,
  backoff 1, 2, 5, 10, then 30 s; one log line per state change; stream `ANFS` created or its
  limits updated; another stream on `anfs.>` refused by name; publish with `Nats-Msg-Id` and
  `Content-Type: application/cloudevents+json`; no reconnect buffer: the files are the buffer);
  `internal/shipper` (batches of at most 256 awaiting acks, a token bucket per scope, a cursor
  moved only over the acknowledged prefix of its file, an invalid line logged once and stepped
  past; every wait blocks on a channel or a timer); `internal/testnats` (an embedded, restartable
  nats-server for tests). `go/test/e2e.bats`; `.github/workflows/ci.yml` and `release.yml`;
  `README.md`, `AGENTS.md` ("anfsd stays thin").
- **core/nats** (new): nats-server v2.15.1 and natscli v0.5.0 pinned by sha256
  (`pins.yml`), fetched into `~/.cache/nats/<version>/`, unpacked into
  `~/.local/share/nats/<version>/`, linked from `~/.local/bin`; `~/.local/state/nats/` (0700:
  `users/{anfsd,admin,reader}` made once, `server.conf` 0600 rendered from `nats.env`, `port`,
  `jetstream/`); the service `io.anfs.nats` / `anfs-nats.service` on `127.0.0.1:47222`,
  monitoring `47223`. `post_remove` unloads and deletes the service and the links, keeps STATE
  and DATA.
- **core/anfsd** (new): the pinned anfsd release (`pins.yml`: anfs-io/runtime v0.1.0, no sums
  until the release exists) or `ANFSD_BIN` (anfsd.local.env, a build from source); the service
  `io.anfs.anfsd` / `anfsd.service` (`Requires=`/`After=anfs-nats.service`, `KillMode=process`);
  **`anfs bus`** (`anfs-bus`): `status` (`SERVICE STATUS DETAIL`: nats, jetstream, anfsd, the
  shipper's lag per scope, linger on Linux; exit 1 when one is not ok; `--json`), `start`,
  `stop`, `restart [nats|anfsd]`, `reload` (SIGHUP), `logs [nats|anfsd] [-f]`, `nats -- <args>`
  (the admin user and URL in the environment only).
- **lib/anfs**: `pins.sh` (`anfs_pin_read`, `anfs_pin_fetch`, `anfs_pin_link`/`_unlink`,
  `anfs_sha256`, `anfs_pin_platform`); `service.sh` (a user's launchd agent or systemd unit:
  `anfs_service_write`, `_load`, `_stop`, `_remove`, `_state`, `_signal`; `listener.sh` left as
  it is); `bus.sh` (the two service specs, NATS's users and `server.conf`, anfsd's binary).
  `events.sh`: `anfs_events_follow <fn> [deadline]` and `anfs_events_follow_init` (events ls
  follows through it; `source.updated` among the completed types). `report.sh`: `wait` waits on
  events. `sources.sh`: `gitsrc_update` emits `source.updated` (host scope, subject the alias,
  `from`, `to`) for each source whose HEAD moved.
- **anfs rpc**: `bus` is a group; `logs` without `-f` is a read.
- Docs: AGENTS.md ("The bus", the XDG exceptions, the lists' rule 5 and keys, report wait,
  Events, Lib Structure, Testing, the layout, the API's grant), README ("The bus"),
  `packages/{nats,anfsd,anfs}/README.md`, the `anfs-bug-report` skill.

## Commits (branch agents-02a, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| docs | `001bf51` | plan.md: the five open questions recorded as decided |
| runtime | `c3fdb13` | all of `go/`, CI and release, README, AGENTS.md |
| core | `3b121f0` | core/nats, core/anfsd, anfs bus, lib/anfs/{pins,service,bus}.sh, report wait on events, source.updated, the docs and tests (anfs 0.2.17 already bumped since its last push; nats and anfsd new at 0.1.0) |
| docs | (this) | this log |

## Decided (the person, at the start)

1. `events ls` reads the files even when NATS is up. 2. The dispatched maintainer fixes and
commits in its job (02b). 3. Stream bounds 90 days / 1 GiB, the files kept forever. 4. `job: none`
allowed until `anfs work` moves to core (02b). 5. NATS on 47222. Recorded in plan.md.

## Decisions made during the work (deviations from the plan, and why)

1. **`lib/anfs/bus.sh`**, not in the plan's file list: the service specs, NATS's rendered config
   and "which anfsd" are needed by both packages' hooks and by `anfs bus restart`; one file keeps
   them from disagreeing.
2. **`pins.yml` has a `bin:` key** (the program's path inside the archive), so `pins.sh` needs no
   per-package code; the anfsd release archive holds `anfsd` at its root.
3. **The reader user's publish permissions** add `$JS.API.INFO`,
   `$JS.API.CONSUMER.DURABLE.CREATE.ANFS.>` and `$JS.API.CONSUMER.MSG.NEXT.ANFS.>` to the plan's
   three: `$JS.API.CONSUMER.*.ANFS.>` does not match those two-token API subjects, so a durable
   pull consumer could not be made or read. Still never `anfs.>` (checked live: refused).
4. **Loading a service that is loaded** (an install, `restart`): launchd keeps the plist it was
   loaded with, so it is `bootout` then `bootstrap` (retried twice, a second apart); systemd is
   `daemon-reload`, `enable`, `restart`. `anfs bus start` leaves a running service alone;
   `stop` is `bootout` / `systemctl stop` (it starts again at the next login or boot).
5. **`anfs bus status`** words: `ok`, `fail`, `stopped`, `not installed`, `lag` (a row per scope
   with unshipped lines), `off` (linger). Every non-`ok` row makes the exit 1, lag and linger
   included, so `status` is 0 only when everything is shipped.
6. **`source.updated` from every `gitsrc_update`** in a tool that sourced events.sh (anfs: `anfs
   src update`, and the automatic updates of `anfs install`), a clone included (`from` empty), not
   only `anfs src update`. ppm's automatic updates emit nothing (ppm does not source events.sh).
7. **`report wait`**: the descriptors are positioned at the files' ends before the state is read
   once, so an event between the two is not lost. A merge with no event is not seen (a test holds
   it: nothing polls git).
8. **Test hooks**: `ANFS_PIN_PLATFORM` (pins.sh) beside `ANFS_SERVICE_OS`; anfsd honors
   `ANFSD_CONFIG_HOME`/`ANFSD_STATE_HOME` from the environment, as the tools do `<TOOL>_*_HOME`.
9. **Pins checks**: `packages/anfs/tests/pins-check <pins.yml> [--print]` (every release platform,
   the program found in the archive), run by `packages/{nats,anfsd}/tests/pins`. Run for nats:
   all eight sums match the plan's.
10. **CONFIG**: the "nothing in CONFIG but the linked `.env`" check is in nats.bats and anfsd.bats
    (after a real `post_install`); xdg.bats's grep already covers the two new `install.sh`.
11. **Go dependencies**: nats.go v1.54.0, nats-server/v2 v2.15.1 (tests only), fsnotify v1.10.1,
    google/uuid v1.6.0, golang.org/x/time v0.16.0 (the token bucket). No YAML parser yet: 02a
    reads no YAML (02b adds one).
12. CI installs bats with apt and fetches the pinned linux-amd64 nats-server and nats CLI,
    checking the same sums; actions are pinned to checkout v7.0.1 and setup-go v7.0.0 by commit.

## Tests

- **core bats** (`anfs work run agents-02a core -- ... </dev/null`): `packages/nats/tests/nats.bats`
  11, `packages/anfsd/tests/anfsd.bats` 7, `packages/anfsd/tests/bus.bats` 11 (all new);
  `report.bats` (wait: at once, on another process's close, on a later `source.updated`, a merge
  with no event times out, `git merge-base` once per event and none per second; `--interval` is
  64), `sources.bats` (+1: `source.updated` only for sources that moved), `events.bats` (+1: follow
  from now, a predicate, a deadline, idle CPU), `rpc.bats` (+1: bus reads and writes),
  `registry.bats` and `settings.bats` updated.
- **The full run, once**: `packages/anfs/tests/all`: **726 passed, 0 failed**, 53 files (pmm and
  pcm off), 56 s (01 ended at 691).
- **Go** (`go test -race ./...` in `runtime/go`): **29 tests**, all passing, three runs with
  `-count=3` and none flaky; gofmt and vet clean. Among them: N events are N messages on the right
  subjects; a crash between ack and cursor still gives N; NATS down at start then up; NATS
  restarted mid-stream loses nothing; the rate limit holds; an invalid line logged once; the
  cursor never ahead of an ack; a foreign stream said once; **an idle anfsd used 32 ms of CPU over
  11 s** (the limit is 200 ms in 10 s).
- **e2e** (`go/test/e2e.bats`, the pinned nats-server and nats CLI fetched into the job's TMPDIR
  and checked): 3 passing (each line once with its id and subject, kill -9 and restart with the
  cursor reset still once each, SIGTERM leaves exit 0, the cursors and no pid file).
- **A live check** in a temp dir (not a test file): the rendered `server.conf` under the real
  nats-server v2.15.1 (`-t` valid; then running), anfsd built from source, three bash-emitted
  events and `anfs bus status` with a stub `launchctl`: before anfsd, `jetstream fail (no stream
  ANFS yet)` and `shipper lag host: 3 lines`; after, all ok, `stream ANFS: 3 messages`; one more
  emit shipped at once; `anfs bus nats -- stream info ANFS` works; the reader user cannot publish
  `anfs.host.x.y` and can read the stream's info.

## Not run

- pmm's and pcm's suites (off). Nothing was installed on this host and no service was loaded: every
  `launchctl`/`systemctl` was a stub, every nats-server and anfsd ran in a temp dir and was
  stopped by pid. The real `~/.local/state` was not written.
- `packages/anfsd/tests/pins`: there is no release yet (it fails on the download, as it should).
- The two idle-CPU asserts in events.bats read `ps -o time=` in macOS's format (as 01's did); on
  Linux they need the other format.
- CI and the release workflow have not run (nothing pushed).

## By hand, for the person after merge

1. **anfsd's release, or from source until then.** Merge the runtime branch, push `main`, then
   tag and push the tag; the release workflow builds and attaches the four archives and
   `SHA256SUMS`:

   ```bash
   cd ~/spaces/anfs-dev/runtime
   git push origin main
   git tag v0.1.0 && git push origin v0.1.0
   gh run watch                                   # the release job
   ```

   Then pin it in core (this session can do it for you): `packages/anfsd/tests/pins --print`,
   paste the four lines under `sha256:` in `packages/anfsd/pins.yml`, `packages/anfsd/tests/pins`
   (all `ok`), commit.

   Until the release exists, build it on this Mac and point core/anfsd at it:

   ```bash
   cd ~/spaces/anfs-dev/runtime/go
   go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
   mkdir -p ~/.config/anfsd
   echo 'ANFSD_BIN=~/.local/share/anfsd/dev/anfsd' >> ~/.config/anfsd/anfsd.local.env
   ```

   (Once the release is pinned: delete that line and `anfs bus restart anfsd`.)

2. **Install**:

   ```bash
   anfs src update core
   ppm install core/nats core/anfsd
   ```

   On a Debian host also `loginctl enable-linger $USER`.

3. **Status**: `anfs bus status`. Every row `ok` (`anfsd ... (from source: ...)` until the
   release); the `shipper` rows show `lag` for a few seconds while 01's history ships, then
   `nothing to ship`. Then `anfs bus nats -- stream info ANFS` shows the messages, and
   `anfs events ls --all --json | wc -l` the same count (it reads the files; lines that are not
   events are not shipped and are in `anfs bus logs anfsd`).

4. **A report reaches NATS**: in one terminal `anfs bus nats -- sub 'anfs.*.report.>'`; in
   another, file a throwaway report the way you want to test it (a file report appends to
   BACKLOG.md: delete the section afterwards; the event stays in the files and the stream, as
   every event does):

   ```bash
   anfs report --title "bus check" --actual "testing the bus"   # prints r-<date>-<n>
   anfs bus nats -- stream info ANFS                            # one more message
   anfs bus nats -- stream get ANFS --last-for 'anfs.*.report.filed'   # the event, as filed
   anfs report close r-<date>-<n> --commit $(git -C ~/.local/share/anfs/sources/core rev-parse HEAD)
   ```

   The subscriber prints the `report.filed`, then the `report.closed`, each on
   `anfs.<space id>.report.<verb>`. `anfs report wait <id>` on a closed report whose commit is
   here returns at once.

5. **Stopping and starting**:

   ```bash
   anfs bus stop                 # anfsd, then NATS (they start again at the next login)
   anfs bus status               # nats and anfsd stopped; exit 1
   anfs report -n --title x --actual y   # emitting is unaffected (the -n files nothing)
   anfs bus start                # NATS, then anfsd; the backlog ships
   anfs bus restart anfsd        # after a new build or an anfsd.env change
   anfs bus logs anfsd           # one line per change of state
   ```

## Open questions

1. `anfs bus status` exits 1 while anything is unshipped (lag) and on Linux without linger; if a
   script should read it as healthy during a catch-up, `lag` could become `ok` with a detail.
2. `ppm install core/anfsd` before the first release needs `ANFSD_BIN`; the person may prefer
   core/anfsd to stay out of any stack until v0.1.0 is pinned (it is in none now).
