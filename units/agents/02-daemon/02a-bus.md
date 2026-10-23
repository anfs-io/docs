# Plan 02a — the bus: core/nats, the shipper, `anfs bus`

Written 2026-10-23 against core main `870eece` and an empty `anfs-io/runtime`. Part of
[plan 02](plan.md), whose decisions 1, 2, 6, 7 and 8 this builds. Mergeable alone: after it,
every event is on a durable bus, and `anfs report wait` no longer polls git.

## Context — read these files first

The job is `anfs work new agents-02a core runtime docs` (runtime is anfs-dev's space repository,
as docs is). Paths are core's unless they name another repo.

- docs: `AGENT-NATIVE.md`, `units/agents.md`, `units/agents/01-events/{plan,log}.md` (the profile),
  [plan.md](plan.md) here
- `AGENTS.md`: "Where files go", "Configuration", "Package Structure" (`install.sh` hooks),
  "The toolkit API" and "The listener", "Events", Testing
- `lib/anfs/listener.sh` and `packages/anfs/install.sh` (the pattern for written units, stubbed
  `launchctl`/`systemctl`), `lib/anfs/events.sh` (the follow loop), `lib/anfs/report.sh`
  (`_anfs_report_state`, `cmd_report_wait`), `lib/anfs/sources.sh` (`anfs src update`)
- stack: `packages/obsidian/home/.local/bin/ovm` (`_sha256`, the pinned fetch) and
  `packages/obsidian/tests/pins`; `packages/go/README.md`
- anfs-dev: `packages/dev/home/.local/bin/anfs-work` (the `anfs-<name>` seam)

## core/nats (new package)

```
packages/nats/
  package.yml        # platforms: [macos, linux]; no brew: the binaries are pinned downloads
  pins.yml           # nats-server and nats: repo, version, asset pattern, sha256 per platform
  install.sh         # post_install: fetch, place, render server.conf, write and load the unit
  home/.config/nats/nats.env
  README.md
  tests/{nats.bats,pins}
```

`pins.yml` (the sums are the releases' own `SHA256SUMS`, 2026-10-23; `tests/pins --print`
re-fetches the assets and checks them before merge):

```yaml
nats-server:
  repo: nats-io/nats-server
  version: v2.15.1
  asset: nats-server-{version}-{os}-{arch}.tar.gz   # holds nats-server-{version}-{os}-{arch}/nats-server
  sha256:
    darwin-arm64: ed0fe39222ec2e317c4d0e1a9cdc16c63ec523d94f9355d24a902a4db2655ffd
    darwin-amd64: 3cba5976a4fbd022669d3401cab4147faeb2b4c079712c882edb9ae4e52feb06
    linux-amd64:  bf5edc7b1ce98e885c368cb43a886eb148dc5d0551d99f3cf6141fb5e1954c70
    linux-arm64:  e2fbab4cad7d3c76e3052eb959570393c9d73ba756b70bf8e9eea61e9ae6d49c
nats:
  repo: nats-io/natscli
  version: v0.5.0
  asset: nats-{version#v}-{os}-{arch}.zip           # holds nats-{version#v}-{os}-{arch}/nats
  sha256:
    darwin-arm64: 37a6f924bc42c72a38856d9c6125c38885732b2a13387b8bc15d1ff80987520b
    darwin-amd64: 9fcf6dbe6462c78a1564d03c8970158f50f50c31ebda5350d2da4a597de796e7
    linux-amd64:  d9fc93e9e9ab0310deff7e719a8a0da6f9d66f26d473f039aaee70155507aecc
    linux-arm64:  39496b0e332ba8d1b7dcbac2019c7d1e7a7f0d0ccefef9da547e9d6eeed408c8
```

- **Fetch** (`lib/anfs/pins.sh`, new, shared with core/anfsd): `anfs_pin_fetch <pins.yml> <name>`
  downloads into `~/.cache/<name>/<version>/` once, checks the sha256 (`sha256sum`, else `shasum
  -a 256`), unpacks into `~/.local/share/nats/<version>/`, and the hook links `~/.local/bin/nats`
  and `nats-server` to it. A mismatch fails the install (`ppm_fail`, both sums named, the file
  deleted); no curl, or an unpinned platform, is `ppm_fail` with what to do. The zip needs `unzip`
  (on macOS; `system: {linux: [unzip]}` in package.yml).
- **Settings** (`nats.env`, linked; `nats.local.env` for this machine): `ANFS_NATS_PORT=47222`,
  `ANFS_NATS_HTTP_PORT=47223`, `ANFS_NATS_MAX_FILE=2GiB` (JetStream's disk cap),
  `ANFS_NATS_MAX_MEM=64MiB`.
- **`~/.local/state/nats/`** (0700): `server.conf` (0600, rendered; never edited), `users/{anfsd,
  admin,reader}` (one password each, 64 hex, made once, kept by a reinstall), `port`, `jetstream/`
  (the store), `nats-server.log` (macOS). `server.conf`: `listen 127.0.0.1:<port>`, `http
  127.0.0.1:<http port>`, `server_name anfs`, `max_payload 64KB`, `jetstream {store_dir, max_file_store,
  max_memory_store}`, `authorization {users: [...]}`:
  - `anfsd`: everything (publishes `anfs.>`, manages the stream and its consumers)
  - `admin`: everything (the person's CLI)
  - `reader`: subscribe `anfs.>` and `_INBOX.>`; publish only the JetStream API for stream `ANFS`'s
    info and its own consumers (`$JS.API.STREAM.INFO.ANFS`, `$JS.API.CONSUMER.*.ANFS.>`,
    `$JS.ACK.ANFS.>`); never `anfs.>`. Unused until 03.
- **The unit** (`lib/anfs/service.sh`, new; decision 7's table): `nats-server -c
  ~/.local/state/nats/server.conf`. `post_remove` unloads and deletes it and the links; DATA, the
  store and the passwords stay (an `anfs implode` removes STATE as today).

## core/anfsd (new package)

```
packages/anfsd/
  package.yml        # depends: [nats]
  pins.yml           # anfs-io/runtime, v0.1.0, anfsd-{version}-{os}-{arch}.tar.gz (sums after the first release)
  install.sh
  home/.config/anfsd/anfsd.env
  home/.local/bin/anfs-bus     # anfs bus ...
  README.md
  tests/{anfsd.bats,bus.bats,pins}
```

- **Placed** as core/nats is: `~/.local/share/anfsd/<version>/anfsd`, linked from `~/.local/bin`.
  `ANFSD_BIN` (in `anfsd.local.env`) overrides it with a binary built from source (plan.md, 6).
- **Settings** (`anfsd.env`): `ANFSD_BIN`, `ANFSD_STREAM_MAX_AGE=90d`, `ANFSD_STREAM_MAX_BYTES=1GiB`,
  `ANFSD_DUP_WINDOW=24h`, `ANFSD_SHIP_RATE=100`, `ANFSD_SHIP_BURST=1000`. anfsd reads both files
  itself (`KEY=value`, `~/` expanded, nothing else), with `anfs_conf_read`'s rules.
- **`~/.local/state/anfsd/`**: `cursors.json`, `anfsd.log` (macOS), `anfsd.pid`.
- **The unit**: `anfsd run`, after nats (decision 7).

## anfsd (runtime, `go/`)

```
go/go.mod                         module github.com/anfs-io/runtime/go
go/cmd/anfsd/main.go              anfsd run | version | ship --once (tests) | help
go/internal/config                anfsd.env, nats's port and the anfsd password, the XDG paths
go/internal/cloudevent            the profile: parse and validate a line (anfsv 1), write one (UUIDv7, O_APPEND, ≤ 4096)
go/internal/spool                 scopes, month files, fsnotify + 30 s reconcile, complete lines, cursors
go/internal/bus                   connect (backoff 1/2/5/10/30 s), ensure stream ANFS, publish with Nats-Msg-Id
go/internal/shipper               spool → bus: window 256, token bucket per scope, cursor after the acked prefix
go/test/e2e.bats
.github/workflows/ci.yml, release.yml
README.md, AGENTS.md              how to build, test, release; the layout; "anfsd stays thin"
```

- `anfsd run`: loads config; ensures `ANFS` (create, or update its limits; another stream already
  holding `anfs.>` is an error in the log and `anfs bus status`); ships (plan.md, 2); SIGTERM
  drains in-flight acks for up to 5 s and writes the cursors. Logs one line per state change (to
  stderr: launchd's file, the journal).
- No HTTP, no RPC: `anfs bus status` reads anfsd's state files and NATS's monitoring port.

## `anfs bus` (core/anfsd's `anfs-bus`)

| Command | Does |
| --- | --- |
| `anfs bus status` | `SERVICE STATUS DETAIL`: `nats` (healthz on the monitoring port; version, port), `jetstream` (`/jsz`: messages, bytes, first and last time), `anfsd` (unit loaded, pid alive, version, `release` or `from source`), `shipper` (per scope with a lag: lines and bytes past the cursor). Exit 0 all ok, else 1. `--json` |
| `anfs bus start\|stop\|restart [nats\|anfsd]` | load, unload, or both (both services when none is named; `restart nats` re-renders `server.conf` from the settings first) |
| `anfs bus logs [nats\|anfsd] [-f]` | the log file (macOS) or `journalctl --user -u` (Linux) |
| `anfs bus nats -- <args>` | runs `nats` with `NATS_URL` and the `admin` user from STATE in its environment, never on its command line |

Under the toolkit API, `status` and `logs` (without `-f`) are reads; the rest writes.

## Retiring `anfs report wait`'s polling

- `anfs src update` emits `source.updated <alias> from=<sha> to=<sha>` into the host scope for
  each source whose HEAD moved.
- `anfs report wait <id> [--timeout S]`: reads the state once (as now: `available` is 0, an unknown
  id 2); then follows the report's scope and the host scope with events.sh's follow loop (no
  `--interval`, removed) and re-reads the state only when a `report.closed` for this id or a
  `source.updated` arrives: one `git merge-base` per event, none per tick. A GitHub issue closed
  by hand emits nothing; `wait` sees it at the next `source.updated` or never (its help says so,
  and `anfs report status` still asks GitHub).
- AGENTS.md, the skill and `report.sh`'s header say: `report wait` waits on events; `wsm events ls
  -f --once --type report.closed --subject <id>` is the bare form.

## Files

- core: `packages/nats/**`, `packages/anfsd/**` (new); `lib/anfs/{service,pins}.sh` (new);
  `report.sh` (wait), `sources.sh` (the emit), `events.sh` (`source.updated` in
  `ANFS_EVENT_TYPES`; the follow loop callable with a predicate); `AGENTS.md` ("The bus", the XDG
  table's new rows and the units among the accepted exceptions, Testing); `README.md`;
  `skills/anfs-bug-report/SKILL.md`; `packages/anfs/tests/{report,events,xdg}.bats`
- runtime: everything above (the first commit)
- docs: this plan's `02a-log.md`

## Test Spec

core (bats, stubs only):
- `nats.bats`: install with a stub `curl` serving fixture archives → binaries in DATA, links in
  `~/.local/bin`; a wrong sum fails with both sums and leaves nothing; an unpinned platform fails;
  `server.conf` is 0600 with three users and the port from `nats.local.env`; passwords survive a
  reinstall; the plist (macOS) and the unit (Linux) written and loaded through the stubs;
  `post_remove` unloads, deletes the unit and the links, keeps STATE.
- `anfsd.bats`: the same for anfsd; `ANFSD_BIN` puts that path in the unit.
- `bus.bats`: `status` rows with nats up and down (stub `curl` on the monitoring port), a lag from
  a fixture `cursors.json` and spool; exit codes; `start`/`stop`/`restart` call the right stub
  commands on both platforms; `nats --` passes the password only in the environment.
- `report.bats`: `wait` returns 0 when another process emits `report.closed` and the commit is in
  the checkout; returns 0 on a later `source.updated`; times out with 1; calls `git merge-base` once
  per event, not per second (a stub `git` counts); `--interval` is 64.
- `sources`: `anfs src update` emits `source.updated` only for sources that moved.
- `xdg.bats`: nothing written into CONFIG but the linked `.env` files.

runtime (Go, embedded server; then e2e):
- `cloudevent`: 01's example line parses; invalid lines (no `specversion`, `anfsv` 2, over 4096,
  non-string data) are refused; written lines are valid, ≤ 4096, one write each, ids UUIDv7 in
  time order.
- `spool`: a partial line waits for its newline; a new month file, a new scope and a late line to
  last month's file are all shipped; cursors survive a restart and are never ahead of an ack.
- `shipper`: N events give N messages on the right subjects with `Nats-Msg-Id`; acked but not
  recorded (a crash between) then restarted still gives N (the window); NATS down at start, then
  up, ships the backlog; NATS restarted mid-stream loses nothing; the rate limit holds; an invalid
  line is logged once and skipped.
- `bus`: stream `ANFS` created with the settings, its limits updated on change; a foreign stream
  on `anfs.>` is reported.
- Idle: anfsd with nothing to ship uses under 0.2 s CPU in 10 s (no loop).
- `e2e.bats`: the real `nats-server` on a random port in a temp dir, anfsd built, a temp `HOME`
  with fixture spool lines; every line is in the stream once (`nats stream info`, `nats stream
  get`); kill -9 anfsd, append, restart: still once each.

## Verification

- While working: `anfs work run agents-02a core -- bats packages/nats/tests </dev/null` (and
  `anfsd`, the `anfs` files touched); `anfs work run agents-02a runtime -- sh -c 'cd go && go test
  -race ./...'`; `anfs work run agents-02a runtime -- bats go/test/e2e.bats </dev/null` (needs a
  `nats-server` on PATH: a binary fetched into the job's TMPDIR with the pinned sum, never
  installed)
- Once, at the end: `anfs work run agents-02a core -- packages/anfs/tests/all </dev/null`
- Nothing installed, no service loaded, the real STATE untouched.

## By hand, for the person after merge

1. Merge runtime's branch, push it, tag `v0.1.0` and push the tag (CI builds and attaches the
   binaries). Then, in core (this session for you): `packages/anfsd/tests/pins --print`, paste the
   four sums into `packages/anfsd/pins.yml`, commit.
2. `anfs src update core && ppm install core/nats core/anfsd`.
3. `anfs bus status`: all `ok`; the shipper's lag goes to 0 (01's history shipped).
   `anfs bus nats -- stream info ANFS` shows the messages; `anfs events ls --all | wc -l` is the same
   count.
4. On Debian hosts: `loginctl enable-linger $USER`.
5. `anfs report wait` on a closed report returns at once.
