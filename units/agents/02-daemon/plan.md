# Plan 02 — the daemon: NATS, the shipper, dispatch, supervision (index)

Written 2026-10-23 against core main `870eece`, stack main, anfs-dev main and an empty
`anfs-io/runtime`. Binding: `AGENT-NATIVE.md` (all of it, "Decided" above all), the unit
(`units/agents.md`), plan 01's profile and its log, and the decisions in the job's brief (NATS +
JetStream native on the host, `core/nats` and `core/anfsd`, anfsd not a broker, strict CloudEvents,
`access:`, what 02 retires, identity, the principles). Nothing below reopens them.

02 is three plans, each merged and tested on its own, in this order:

| Plan | What | Repos |
| --- | --- | --- |
| [02a — the bus](02a-bus.md) | `core/nats` (pinned binaries, config, service); `core/anfsd` with only the shipper (spool → JetStream); `anfs bus` (status, start, stop, logs, the CLI with credentials); the runtime repo, its CI and release; `lib/anfs/service.sh`, `lib/anfs/pins.sh`; `anfs src update` emits `source.updated`, and `anfs report wait` waits on events (the git polling goes) | core, runtime, docs |
| [02b — dispatch](02b-dispatch.md) | `dispatch:`, `agents:` and `access:` in `space.yml`; the dispatcher and the supervisor in anfsd; `anfs agents` (ls, show, logs, stop, rules); `run.*` events; the report sink (BACKLOG.md becomes a view; report ids unique per space); anfs-dev's rule that retires the grep watcher | core, runtime, anfs-dev, docs |
| [02c — podman events](02c-podman.md) | anfsd follows `podman events` and emits `container.*` and `pod.*` into the host scope (03 needs them) | runtime, core (docs, types), docs |
| [02d — safe to turn on dispatch](02d-safe.md) | after the review (F1, F5, F6, F7): a per-run spool anfsd ingests and stamps with the run's identity (`actorrun`, `actorclaimed`); runs get an environment allow-list; dispatched claude loads no user or project settings (`--setting-sources local --strict-mcp-config`, `--bare` opt-in); `queue` and `rate` hold instead of dropping (`overflow:`, `hold:`, `anfs agents redispatch`); `dispatcher: none` per space. Then anfs-dev's rules may be turned on | runtime, core, docs |

Why three: 02a is useful alone (every event is on the bus, durable, and `report wait` no longer
polls), 02b is the person's main review point and depends on a running bus, and 02c is small and
independent of dispatch. One plan would be one unreviewable branch across four repos. Two (02a+c,
02b) was considered: podman's reconnect logic against a podman machine that comes and goes is its
own test surface, and keeping it out of 02a lets 02a merge first.

## The decisions (each with the recommendation, and the alternatives in a line)

### 1. Subjects, the stream, consumers, readers

- **Subjects: `anfs.<scope>.<noun>.<verb>`**, the scope a space id (10 of `a-z0-9`) or `host`, the
  rest the event's short type (`io.anfs.` dropped): `anfs.tkj76y3nfx.report.filed`,
  `anfs.host.container.started`. "host" cannot be an id (4 characters). A type with more segments
  keeps them. Filters: `anfs.<id>.>` (a space), `anfs.*.report.>` (a kind everywhere).
  *Alternatives:* `anfs.<type>.<space>` (type first: worse for per-space permissions later, which is
  what `accepts:` will want); a stream per space (more streams, no gain on one host).
- **One stream, `ANFS`**, subjects `anfs.>`, file storage in `~/.local/state/nats/jetstream/`,
  retention `limits`, discard `old`, `max_age` 90 days and `max_bytes` 1 GiB (both in `nats.env`),
  duplicate window **24 h** (not the 2 min default: see 2). The JSONL files stay the complete,
  permanent record; the stream is a bounded window for programs. This answers AGENT-NATIVE's open
  question 1 (retention) for the bus; the files are kept as before. *Alternatives:* unbounded (the
  docs warn a stream with no limit fills the disk); interest or work-queue retention (loses history
  that late subscribers need).
- **Consumers: durable, one per subscriber, named for it** (`anfsd-dispatch`, `anfsd-sink-report`,
  03's `ui`), explicit ack, filter subjects narrowed to what the subscriber handles (NATS ≥ 2.10
  takes several filters), deliver policy `new` when first created (a new subscriber never acts on
  history). *Alternatives:* ephemeral ordered consumers (lose the cursor across restarts: against
  "durable before real time"); a KV of cursors kept by us (reinvents consumers).
- **Readers: `wsm events ls` and `anfs events ls` keep reading the files, `-f` included, whether
  NATS is up or not.** The files are written first (an emit never waits on the bus), so on this host
  they are never behind the stream; the stream is bounded, the files are not; and one code path is
  one thing to test. NATS is for programs: anfsd's consumers, the ui in its container (03), bridges
  later. The person looks at the bus itself with `anfs bus nats -- sub 'anfs.>'` or `-- stream info
  ANFS`. *Alternative:* read NATS when up and fall back to the files (push instead of a 1 s tick,
  but two paths that can disagree for the same command; decided 1).

### 2. The shipper (02a)

Tails `$XDG_STATE_HOME/anfs/events/*/*.jsonl`, sends each complete line as it is, records how far
it got. In anfsd, `internal/spool` and `internal/shipper`.

- **Finding lines:** fsnotify (kqueue on macOS, inotify on Linux) on `events/` and each scope
  directory, plus a reconcile scan every 30 s; no other timer. A partial line (no `\n` yet) waits.
  Every month file keeps its own cursor, so a line written to last month's file after the new one
  appeared is shipped too (01's accepted gap is closed for the bus).
- **Cursors:** `~/.local/state/anfsd/cursors.json`, `{"<scope>/<YYYY-MM>.jsonl": <byte offset>}`,
  rewritten (temp file, fsync, rename) after each batch, only up to the contiguous prefix of lines
  NATS acknowledged. Startup reads them; a file with no cursor ships from its start (the first start
  ships 01's history).
- **Exactly once per event id:** each publish carries `Nats-Msg-Id: <CloudEvent id>`, and the
  stream's duplicate window is 24 h. A crash between NATS's ack and the cursor's write resends at
  most one batch, which the window drops; an outage longer than 24 h with such a batch in flight
  could duplicate it, so every consumer is also idempotent by event id (dispatch records one run
  per rule and event id; the sink skips a report already rendered). Headers also carry
  `Content-Type: application/cloudevents+json`.
- **Validation:** a line that is not a CloudEvent of a known `anfsv` (01's rules) is not sent; it
  goes to anfsd's log with its file and offset, once, and the cursor moves past it.
- **Ordering:** in file order per scope (the order lines were appended); across scopes none is
  promised (ids sort by time to the millisecond, which is what readers use).
- **Backpressure:** at most 256 publishes awaiting acks; a token bucket per scope (100 events/s,
  burst 1000, in `anfsd.env`). Above that the shipper waits, and the files hold the rest: emitting
  never slows down.
- **NATS down:** reconnect with backoff 1, 2, 5, 10, 30 s (then every 30 s), one log line per
  state change, never a tight loop. Emitting is unaffected; the cursors resume when it is back.
  `anfs bus status` shows the lag (bytes and lines not yet shipped, per scope).
- **Who publishes:** only anfsd (the `anfsd` NATS user). The tools write files; anfsd's own events
  (`run.*`, `container.*`) also go through the spool with a Go writer of the same profile (one
  `O_APPEND` write, ≤ 4096 bytes), so the files stay the mirror of everything.

### 3. The dispatch rule format (02b; the main review point)

In the space's `space.yml`, beside 01's `id:`. `agents:` names the roles the space runs; each role
is the address `<space path>/<role>`. `dispatch:` says which event starts which role. The full
format and the semantics are in [02b](02b-dispatch.md); in short:

```yaml
access:                                    # what this space's dispatched agents may reach beyond their job
  - {path: ~/.local/share/anfs/sources, mode: read}
agents:
  maintainer:                              # anfs-dev/maintainer
    harness: claude                        # claude | opencode | pi | antigravity
    concurrency: 1                         # runs of this role at once (more wait in a queue)
dispatch:
  - name: fix-core-report
    on: {type: report.filed, data: {to: file, package: "core/*"}}
    as: maintainer
    job: {name: "fix-{{subject}}", repos: [core]}   # anfs work new; none: the space's directory, read-only
    prompt_file: prompts/report-filed.md   # or prompt: | ...  ({{subject}}, {{data.title}}, ...)
    timeout: 2h
```

- The event matches on `type` (a glob, short form), optional `subject` (glob), `data` (globs per
  key), and `space` (default: this space only).
- Each run: queued (`run.submitted`), started in its job with `ANFS_ACTOR=<harness>:<run id's
  last 8>` and `ANFS_AS=<space path>/<role>` (`run.working`), and ends `run.completed`,
  `run.failed` (exit, timeout, job, spawn, orphaned), `run.canceled` (`anfs agents stop`) or
  `run.rejected` (queue full, rate, stale, invalid). These are A2A's task states, without
  `input-required` and `auth-required`, which are 04's approvals.
- A failed run is not retried (`retries: 0`, settable to 2 at most): it is reported. The person sees
  it in `anfs agents ls` and `anfs events ls --type 'run.*'`.

**anfs-dev's rule** (replacing the grep watcher) is two rules, `fix-core-report` and
`fix-stack-report` (`package: "core/*"` into a core job, `"stack/*"` into a stack job). Until
04's approvals exist, the dispatched maintainer may, inside its job only: read the report (`anfs
report show`), reproduce the bug, fix it with tests, run them through `anfs work run`, commit on
the job's branch, and end with a summary (carried by `run.completed`). It may not push, merge, install (`ppm`,
`anfs install`, `wsm install`), close the report, call `gh`, or write outside the job and its
`access:` (the denials are passed to the harness, 5). Merge, install, push and `anfs report close`
stay with the person, or with this session acting for them. *Alternatives:* triage only (reproduce
and write a plan, no fix: safer, half the value; decided 2); a rule that only posts to an
inbox for a running session (that is 04's inbox).

### 4. Supervision (02b)

- **Runs are children of anfsd**, each in its own session (`setsid`), cwd the job (or the space),
  stdin `/dev/null`, stdout and stderr to `~/.local/state/anfsd/runs/<run id>/out.log`; the record
  `run.json` beside it (rule, space id, trigger event id, role, harness, argv, pid, pgid, start
  time, state, exit, job). Run ids are UUIDv7.
- **Timeout:** SIGTERM to the process group, SIGKILL 30 s later; `run.failed reason=timeout`.
- **Restart policy:** a run is a task, not a service: it is never restarted (`retries:` aside).
  anfsd itself is restarted by launchd (`KeepAlive`, `ThrottleInterval` 10) or systemd
  (`Restart=on-failure`, `RestartSec=5`), and stopping or restarting anfsd **does not stop runs**
  (their own sessions; systemd `KillMode=process`): on start anfsd adopts every `working` run whose
  pid is alive with the recorded start time (checked every 10 s, only while one is adopted), and
  marks the others `run.failed reason=orphaned`.
- **`anfs agents`** (core/anfsd's `anfs-agents`, the package seam `anfs work` uses): `ls` (`RUN
  STATUS RULE ROLE SPACE HARNESS JOB AGE`, read from the run records, so it works with anfsd down),
  `show <run>`, `logs <run> [-f]`, `stop <run>|--all` (writes a `cancel` marker, then signals the
  group: anfsd reports `run.canceled`), `rules [SPEC]` (`RULE STATUS SPACE ON AS`, `invalid` with
  the reason; parsed by `anfsd rules --json`, one parser).

### 5. `access:` per harness (02b)

`access: [{path, mode: read|write}]`; `~` and `$VAR` expanded, a relative path from the space's
directory. The run always gets its job (write) and the space's directory (read). Checked against
the CLIs installed here on 2026-10-23:

| Harness | Version here | How access is handed over | Enforced? |
| --- | --- | --- | --- |
| claude | 2.1.296 | `--add-dir` per path; a per-run `--settings` file: `Read(//p/**)` allow for read, `Edit`/`Write(//p/**)` allow for write, deny rules for the forbidden commands (`Bash(git push:*)`, `Bash(ppm install:*)`, ...), `sandbox.enabled` so Bash writes are confined by the OS (Seatbelt, bubblewrap); `--permission-mode dontAsk` (anything not allowed is denied, never asked) | Yes for its tools and, with the sandbox, Bash writes. Bash reads outside are not confined. To verify on 2.1.296 at 02b's start |
| opencode | 1.18.35 | `OPENCODE_CONFIG_CONTENT` with `permission.external_directory` (`<p>/**: allow`, `*: deny`) and `permission.bash` denials; `opencode run --dir <job>` | Partly: one permission covers read and write outside the project (no read-only), and its own text says bash path checks are "advisory only" |
| pi | 1.1.0 | No permission system. `--tools` can drop `edit,write` for a read-only rule; the prompt states the access | **No.** Use pi only for rules whose job is its own worktree and whose access is advisory |
| antigravity (`agy`) | 1.3.3 (macOS only) | `agy -p --add-dir <p> --sandbox --print-timeout <t>` | Partly: headless runs deny what needs approval and "settings allow-rules do not apply"; no read/write split. `agy --version` hung here (2 min, no output): checked at 02b's start |

anfsd never passes a skip-permissions flag (`--dangerously-skip-permissions`, `--auto`), and
refuses a rule that asks for one. `links:` stays for people and is not access.

### 6. Packaging and release (02a)

- **`anfs-io/runtime`**: `go/` is one module (`github.com/anfs-io/runtime/go`), `cmd/anfsd`,
  `internal/{cloudevent,spool,shipper,bus,config}` (02a), `internal/{space,dispatch,supervise,harness,sink}`
  (02b), `internal/podman` (02c); `go/test/e2e.bats`; `.github/workflows/{ci,release}.yml`;
  `README.md`, `AGENTS.md`. Dependencies: `nats.go` v1.54.0 (its `jetstream` package; v1.55.0 is
  from today, too new to pin), `nats-server/v2` v2.15.1 (tests only, embedded), `fsnotify`,
  `google/uuid` (v7), a YAML parser. Go from `go.mod`'s `go` line (stack/go has go1.27.3).
- **CI** (`ci.yml`, push and PR): `gofmt -l`, `go vet`, `go test -race ./...`, and the e2e bats
  with the pinned nats-server. **Release** (`release.yml`, a `v*` tag): four builds
  (`darwin`/`linux` × `arm64`/`amd64`, `CGO_ENABLED=0`, `-trimpath`, version stamped),
  `anfsd-<tag>-<os>-<arch>.tar.gz` each and `SHA256SUMS`, attached by `gh release create` with the
  default `GITHUB_TOKEN` (`permissions: contents: write`). Actions pinned by commit sha. The repo
  is public, so downloads need no token. The release tag is the version (`v0.1.0`): the module has
  only `internal/` packages, so no `go/v…` module tags are needed.
- **Pinned in core** as ovm pins its plugins: `packages/<pkg>/pins.yml` (repo, version, the asset
  name pattern, a sha256 per platform), a shared `lib/anfs/pins.sh` (`anfs_pin_fetch`: curl into
  `~/.cache/<tool>/<version>/`, sha256 checked, a mismatch fails the install with both sums), and
  `packages/<pkg>/tests/pins` (fetches the real assets and prints their sums; by hand when bumping,
  never in bats). Binaries unpack into DATA (`~/.local/share/nats/<version>/`,
  `~/.local/share/anfsd/<version>/`) and are linked from `~/.local/bin`. A curl download carries
  no quarantine flag, and Go signs darwin/arm64 binaries ad hoc, so macOS runs them.
- **Building from source on this Mac:** `go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd`
  in the runtime checkout (or a job's worktree), then
  `ANFSD_BIN=~/.local/share/anfsd/dev/anfsd` in `~/.config/anfsd/anfsd.local.env` and `anfs bus
  restart anfsd` (the unit is rewritten with that path; `anfs bus status` says `from source`).
  Unset it to go back to the pinned release.

### 7. Service management (02a)

| | macOS (launchd, `gui/<uid>`) | Debian (systemd `--user`) |
| --- | --- | --- |
| nats | `~/Library/LaunchAgents/io.anfs.nats.plist` | `~/.config/systemd/user/anfs-nats.service` |
| anfsd | `io.anfs.anfsd.plist` | `anfsd.service` (`Requires=`/`After=anfs-nats.service`, `KillMode=process`) |
| Restart | `KeepAlive` true, `ThrottleInterval` 10 | `Restart=on-failure`, `RestartSec=5` |
| Logs | `~/.local/state/nats/nats-server.log`, `~/.local/state/anfsd/anfsd.log` | the journal (`journalctl --user -u ...`) |

Written (absolute paths, the install's `PATH`) by the packages' `post_install`, removed by
`post_remove`, by a new `lib/anfs/service.sh` (the listener's pattern for long-running services;
`listener.sh` is left as it is). NATS listens on `127.0.0.1:47222` (not 4222, which apps in
containers publish), monitoring on `127.0.0.1:47223`. A container reaches host loopback on macOS,
so NATS has users: `anfsd` (publishes), `admin` (the person, through `anfs bus nats`), `reader`
(subscribe and its own consumers only; 03 hands it to the ui). Passwords and the generated
`server.conf` are STATE (`~/.local/state/nats/`, 0600 in 0700); the person's settings are
`~/.config/nats/nats.env` (linked, documented) and `nats.local.env`.

`anfs bus` (core/anfsd's `anfs-bus`): `status` (`SERVICE STATUS DETAIL`: nats, jetstream, anfsd,
shipper lag; 02b adds dispatch and the harnesses, 02c podman; exit 1 when one is not ok), `start`,
`stop`, `restart [nats|anfsd]`, `reload` (SIGHUP: rules), `logs [nats|anfsd] [-f]`, `nats -- <args>`
(the nats CLI with the admin user and the URL). On Debian, running without a login needs
`loginctl enable-linger` (by hand; `status` says so).

### 8. Testing

- **core's bats** (packages `nats`, `anfsd`, `anfs`): `launchctl`, `systemctl`, `curl` and `nats`
  stubbed (curl serves fixture tarballs whose sums the test pins); never a real service, never the
  network. `ANFS_SERVICE_OS` picks the platform, as `ANFS_RPC_OS` does.
- **Go tests** in runtime: an embedded nats-server (`port -1`, JetStream in `t.TempDir()`), so the
  shipper, the consumers and dedup are tested against real JetStream; fake harnesses and a fake
  `podman` as scripts in `testdata/`.
- **One e2e bats in runtime** (`go/test/e2e.bats`): the pinned `nats-server` binary on a random
  port (`-p -1 --ports_file_dir`) with a temp store, the built anfsd, a temp `HOME`. Run in CI and
  in the job; skipped with a message when no `nats-server` is on PATH.
- So: stubs on the bash side, a throwaway real server on the Go side. *Alternative:* a throwaway
  server in core's bats too (core's suite would need the binary on every host that runs it).
- No VM tests.

### 9. Slicing

As the table at the top. Each plan's job and its by-hand steps are in its file.

### 10. By hand, and the decisions

By-hand steps are listed per plan (02a: the first release and its pins, install, check the
backlog shipped; 02b: install, the rule live, the grep-watching session ended; 02c: a container
started shows as an event). Old reports in BACKLOG.md are not migrated (no migration code): they
stay as text, and are closed by hand.

**Decided (the person, 2026-10-23, at 02a's start; each the recommendation):**

1. `events ls` (wsm and anfs) keeps reading the files, `-f` included, even when NATS is up.
2. Before 04, the dispatched maintainer fixes with tests and commits in its job (not triage only).
3. Stream bounds: `max_age` 90 days and `max_bytes` 1 GiB; the JSONL files are kept forever.
4. `job: none` is allowed (cnfs's rules) until `anfs work` moves to core (a later plan).
5. NATS listens on `127.0.0.1:47222` (monitoring `47223`), not the default 4222.

## What 03 (the ui) gets from 02

anfs-ui's `poller.rb` and `events.rb` follow `anfs.log` start lines, or the API's `watch`, and
re-run whole lists. From 02 it has: the `reader` NATS user and `~/.local/state/nats/port` (03 adds
pcm's `bus: read` key to hand them to a container, as `toolkit:` hands the API; on Linux, how a
rootless container reaches host loopback is 03's to settle); a durable consumer of its own
(`ui`) on `ANFS`; `container.*` and `pod.*` (02c), `report.*`, `run.*`, `source.updated`. Events
for the toolkit's other changes (a package installed, a space installed) are not in 02: 03 names
the ones it needs, and the toolkit API's `watch` stays until then.

## Sources checked (2026-10-23, read-only)

- nats-server v2.15.1 (2026-10-09): https://github.com/nats-io/nats-server/releases/tag/v2.15.1,
  https://github.com/nats-io/nats-server/releases/download/v2.15.1/SHA256SUMS
- natscli v0.5.0 (2026-09-17): https://github.com/nats-io/natscli/releases/tag/v0.5.0,
  https://github.com/nats-io/natscli/releases/download/v0.5.0/SHA256SUMS
- nats.go releases (v1.54.0 2026-09-18; v1.55.0 2026-10-23): https://github.com/nats-io/nats.go/releases
- release metadata: https://api.github.com/repos/nats-io/{nats-server,natscli,nats.go}/releases;
  anfs-io/runtime is public: https://api.github.com/repos/anfs-io/runtime
- NATS config (includes only relative to the main file; variables from the environment; port -1):
  https://docs.nats.io/running-a-nats-service/configuration
- JetStream streams (duplicate window 2 min default, `Nats-Msg-Id`, limits retention, discard old,
  one stream per subject): https://docs.nats.io/nats-concepts/jetstream/streams
- A2A 1.0.0 task states (submitted, working, input-required, auth-required, completed, failed,
  canceled, rejected): https://a2a-protocol.org/latest/specification/
- The harnesses: `claude --help`, `opencode run --help` (and its binary's strings for
  `OPENCODE_CONFIG_CONTENT`, `external_directory`), `pi --help`, the strings of `agy` 1.3.3
  (`--add-dir`, `--sandbox`, `--print-timeout`, headless denials), on this Mac.
