---
plan: agents/06-coordinator (06c)
job: agents-06c
date: 2026-10-24
---

# Log: agents/06c, OS notifications from anfsd, and each space's coordinator in the ui

Built on runtime `dc510d3` (06b) and core `0eb40c8` (06b2), both unpushed. Section 6 of the plan is superseded by 06b2; nothing here depends on it.

## What changed

**runtime** (anfsd):

- `internal/notify` (new): the sink, a durable consumer `anfsd-notify` on `anfs.*.anfs.approval.>` plus each other type a space's `notify:` lists (`anfs.*.<type>`; the space is checked in Go), deliver new, explicit ack.
- An `approval.requested` whose `to` is `person` joins its space's batch; any other `approval.*` (given, refused, expired, done, failed) with the same subject drops it. A type the space lists joins too.
- The first event of a space opens a 10 s window. At its end, what is left is one notification: one item is `<space>: <what>` (`anfs-dev: merge core +3, stack +1 (r-20261024-2)`, `install core/pcm, core/anfs`, `push core: origin main`, `policy approvals: ...`, or `run.rejected r-9` for a listed type); several are `<space>: N approvals waiting` (or `N waiting` when mixed). The message is `anfs inbox` (or `anfs events ls` with no approval in it).
- At most one per space per minute: a window ending inside the minute waits for its end, and what comes meanwhile joins that batch.
- Messages are acked when their notification was sent (or dropped, or the item answered), so a restart inside the window delivers them again. `AckWait` 5 min, `MaxAckPending` 1000; a redelivery is folded into its item by event id.
- The notifier, a fixed argv run with `exec` (never a shell), 10 s timeout: `ANFSD_NOTIFY_CMD` (words split on blanks, `{{title}}` and `{{message}}` replaced inside each word); else on macOS `terminal-notifier -title T -message M -group anfs-<space id>` when on PATH, else `osascript -e 'on run argv' -e 'display notification (item 2 of argv) with title (item 1 of argv)' -e 'end run' T M`; on Linux `notify-send -a anfs T M` when it exists and the session bus does (`DBUS_SESSION_BUS_ADDRESS`, or `$XDG_RUNTIME_DIR/bus`).
- `$ANFSD_STATE_HOME/notify.json`: `{"state":"ok|off|unavailable|failing","via","since","error","day","sent"}`; `sent` counts the host's local `day`. Written at start (a probe: which notifier there would be, nothing sent), at each send, and on off. One log line per change of state (`notify: terminal-notifier`, `notify: unavailable (no notify-send)`, `notify: failing (...)`).
- `internal/space`: `notify:` is a list of event types, short or full (`approval.requested`, `io.anfs.run.rejected`, `cnfs.partitia.report.filed`), normalised and deduplicated. Absent: `[approval.requested]`. `[]`: none. Not a list, a glob, or not a type: `NotifyErr`, and the default (the sink logs it once per space).
- `internal/config`: `ANFSD_NOTIFY=on|off` (on), `ANFSD_NOTIFY_CMD` (must hold `{{title}}` or `{{message}}`).
- `cmd/anfsd`: the sink follows each load of the spaces with the lander (`Loaded` calls both), its filters updated when they change; with `ANFSD_NOTIFY=off` no consumer, `notify.json` says off.
- `internal/testfake`: `fake-notify` (records its name and argv, NUL-separated, one file per call; fails while told), linked as `notify`; `Env` sets `ANFSD_NOTIFY_CMD` to it, so an anfsd a test starts never finds `/usr/bin/osascript`. `go/test/e2e.bats` runs anfsd with `ANFSD_NOTIFY=off`.
- README ("OS notifications"), AGENTS.md (layout, the test rule).

**core**:

- `packages/anfsd`: `anfs bus status` gains a `notify` row after podman: `ok  terminal-notifier, 3 sent today` (a count of another day reads 0), `off  ANFSD_NOTIFY=off (anfsd.local.env)`, `unavailable  no notify-send` (both soft: they do not fail the exit), `fail  <error> (N sent today)` for failing, `fail` with no `notify.json` while anfsd runs, `stopped` when it does not. `anfsd.env` documents `ANFSD_NOTIFY` and `ANFSD_NOTIFY_CMD`. README: a "Notifications" section and the settings. Tests: `bus.bats` (rows shifted; a new notify test), `anfsd.bats` (the settings list).
- `packages/anfs`: `lease` joins the toolkit API's `RPC_GROUPS`, so `wsm lease list` is a read (the ui's grant does not need write for it) and `claim`/`renew`/`release` are writes. `rpc.bats`: one test.
- `containers/ui` (0.3.3, the hook): `lib/anfs/coordinators.rb` (new) keeps each space's coordinator. Spaces is listed with `wsm lease list` (at start, every 5 minutes, after a command run from here), and `lease.claimed|released|expired` with subject `coordinator` change it in between with no command: claimed is the actor, `working`; released is free; expired is free until the taker's claimed. The Spaces table has a Coordinator column (`claude:5a52b1c0 (listening)`, `–` when free or no lease); sortable like every column. Events that came while a list ran are applied again over it (ContainerRows's pattern). A failed lease list keeps the last one. `Poller` now maps a type's noun to the rows it updates (`container`, `lease`).
- AGENTS.md: `notify:` among anfsd's `space.yml` keys, "Notifications" under "The bus", the `notify` row of `anfs bus status`, the ui's lease events.

## Commits (branch agents-06c, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| runtime | `470ec5a` | the notify sink, `notify:`, the settings, the fake notifier, tests, README, AGENTS.md |
| core | `f01c231` | anfsd: the notify row, anfsd.env, bus.bats, README |
| core | `0541ed4` | anfs: `lease` a toolkit API group; rpc.bats |
| core | `9419332` | ui: the Coordinator column; specs; README (ui 0.3.3) |
| core | `52b0dfe` | AGENTS.md |
| core | `547a23a` | anfsd.bats: the two settings |
| docs | (this) | this log |

Versions: containers/ui 0.3.2 → 0.3.3 (the hook). core/anfsd 0.3.3 and core/anfs 0.2.19 were already bumped in this unpushed series (the hook said so). Runtime has no version (a tag is the release).

## Decisions made during the work

1. **The consumer takes every `approval.*`**, not only `.requested`: the window needs the answers to drop what `auto` (or a `--through` given before) answered at once. Its filters are `anfs.*.anfs.approval.>` plus `anfs.*.<type>` per extra listed type, never overlapping (approval types are covered by the first).
2. **Only `to: person`** requests notify (every one in 04; a role address later is that role's business).
3. **Ack after the notification**, not on receipt: at-least-once across a restart in the window or the minute. A restart after a notification acked nothing is lost; one before it can notify again.
4. **An invalid `notify:` keeps the default** (approval.requested), logged once: a typo should not silence approvals. `[]` is the only way to none.
5. **Exact types, no globs**, in `notify:`: each is a consumer subject; a glob is refused (invalid, the default).
6. **A scope anfsd does not know** (host for `anfs inbox request`, a space not installed) gets the default; its name in the title is the scope (`host: merge core +1`).
7. **`unavailable` is soft** in `anfs bus status` (a server with no desktop), as podman's `away`; **`failing`** (the notifier ran and failed, usually a bad `ANFSD_NOTIFY_CMD`) is a fail.
8. **The notifier is resolved at each send**, not once: `brew install terminal-notifier` or a session bus that appears later is used without a restart; the state line changes and is logged when it does.
9. **`notify.json`'s fields** never empty in the bus row (`yq ... // "-"`): `read` with a tab IFS joins two tabs into one, which shifted the fields in a first version.
10. **The ui's column key is `holder`**: `wsm ls --json` already has `coordinator` (the recorded policy's state, 06b2). Only the `coordinator` role is shown.
11. **`lease` as a read group** in the toolkit API (not in the plan's file list): the ui holds a write grant and would have worked without it, but a list must be a read.
12. **`ANFSD_NOTIFY_CMD` has no quoting**: words split on blanks; an argument with a space cannot be written. A value without `{{title}}` or `{{message}}` is refused at start (anfsd does not run).

## Tests

Every run was `anfs work run agents-06c <repo> -- <iso> <cmd> </dev/null`. `<iso>` (in the job's TMPDIR) runs the command under `env -i` with a fresh HOME, every `XDG_*` (`XDG_RUNTIME_DIR` too) and TMPDIR under the job's TMPDIR, so every `ANFS_*_HOME`, `WSM_*_HOME`, `WSM_SPACES_HOME` and `ANFSD_STATE_HOME` derives from them; PATH is the system's plus links to go, gofmt, bats, yq, bash, git, jq, nats-server, nats, ruby, bundle, node, zsh; `GOPROXY=off`, GOCACHE and the bundle in the job's TMPDIR. `ANFS_WORK` was never set. No real notifier, claude, harness or podman was run.

- **runtime**: `gofmt -l`, `go vet ./...` clean; `go test ./...` every package ok; `-race -count=2` on notify, space and config ok; `go/test/e2e.bats` **7 passing**.
  - `notify`: one notification per space in the window (two requests: "2 approvals waiting"; an auto pair, `to` not person, a `notify: []` space, an unlisted type: none; a space's listed `run.rejected` notified; an answered request in another space dropped; the consumer's config; every message acked; notify.json ok, 2 sent; one log line); one a minute (the second waits for the minute's end and counts the third); no shell (a title with quotes, `$(...)`, backticks and `;` arrives intact as osascript's argv and as `ANFSD_NOTIFY_CMD`'s, nothing runs); which notifier (terminal-notifier before osascript, notify-send with `DBUS_SESSION_BUS_ADDRESS` or the runtime dir's bus, none without, another OS, the command winning, the command not found); unavailable logged once, failing logged once then ok again, off; the filters following a change of `notify:` and an invalid one logged once.
  - `space`: `notify:` absent, `[]`, normalised and deduplicated, a layer's type; four invalid forms keep the default.
  - `config`: both settings from anfsd.local.env, the defaults, two refusals.
  - `cmd/anfsd`: a started anfsd has `anfsd-notify` with `anfs.*.anfs.approval.>` and notify.json `ok` via the fake `notify`; with `ANFSD_NOTIFY=off`, no consumer, notify.json off, the log line.
- **core**:
  - `packages/anfsd/tests` **45 passing** (bus.bats 16, one new).
  - `rpc.bats` 39 passing (one new).
  - ui: `bundle exec rspec` **106 examples, 0 failures** (6 new: the column rendered; the lease list with Spaces; claimed/released from events with no command; other roles, unknown spaces and other events change nothing; the list wins over older events and one during it is applied again; a failed lease list keeps the last and publishes nothing), with nats-server and node on PATH; `bundle exec rubocop` 38 files, no offenses. The gems went into the job's TMPDIR (`BUNDLE_PATH`), not the host's Ruby; no podman.
  - `packages/anfs/tests/all` (pmm and pcm off): a first run **835 passed, 1 failed** (`anfsd.bats` 7: the settings list in anfsd.env; fixed in `547a23a`), then **836 passed, 0 failed**, 60 files, 129 s.

## Not run

- Linux (notify-send, the session bus, the notify row under systemd). The code paths are covered with the fake on a "linux" OS value only.
- A real notification on macOS (by the rules): terminal-notifier's `-group` and osascript from a LaunchAgent are the plan's claims, unverified here.
- Nothing installed; the running bus, anfsd and ui were not touched.

## By hand, for the person (no migration code)

1. Merge: ff-merge `agents-06c` in runtime, core and docs (after 06b and 06b2; rebase if main moved).
2. Rebuild and restart anfsd: `cd ~/spaces/anfs-dev/runtime/go && go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd`, then `anfs bus restart anfsd`.
3. `anfs src update core`; `ppm install core/anfs core/anfsd` (the toolkit API's `lease` group, the bus row, anfsd.env).
4. Optional: `brew install terminal-notifier` (its look, and one group per space); without it osascript is used. Nothing to restart: the notifier is found at each send.
5. `anfs bus status`: `notify ok terminal-notifier, 0 sent today` (or `osascript`). The first notification from osascript may ask macOS for permission (System Settings, Notifications, Script Editor); from terminal-notifier, for its own.
6. The next approval request pops one within about 10 s. To try it without a run: `anfs inbox request merge --job <a job worked in by hand>` (host scope: the title says `host`), then answer or dismiss it.
7. ui: `pcm down ui && pcm up ui` (lib/ changed: a restart). Spaces shows a Coordinator column; start a Claude session in anfs-dev and it reads `claude:<session> (working)` within 2 s, `(listening)` after the next 5-minute list.
8. On a Linux host: `command -v notify-send` and `systemctl --user show-environment | grep DBUS`; `anfs bus status` says `unavailable` and why otherwise.
9. To silence a space: `notify: []` in its `space.yml` (your commit), `anfs bus reload`. For the host: `ANFSD_NOTIFY=off` in `~/.config/anfsd/anfsd.local.env`, `anfs bus restart anfsd`.

## Open questions

1. **A consumer left by `ANFSD_NOTIFY=off`**: off creates no consumer, but one made while it was on stays in the stream; turned on again, it delivers everything since, and the window drops what was answered meanwhile, so only approvals still waiting pop. **Recommendation: accept** (that is the right result). `anfs bus nats -- consumer rm ANFS anfsd-notify` clears it by hand.
2. **`working` until the next list** in the ui: a renewal is not an event, so a listening coordinator reads `working` for up to 5 minutes after its claim. **Recommendation: accept** (principle 7: keep-alives emit nothing); the next list corrects it.
3. **`ANFSD_NOTIFY_CMD` without quoting**: an argument with a space is impossible. **Recommendation: accept** until a real phone channel needs one; then a JSON argv (`["curl","-d","{{message}}",...]`).
4. **Host-scope requests** (`anfs inbox request`) are titled `host: ...`. **Recommendation: accept**; or title them `anfs:` when the person finds `host` odd.
5. **A notification for each extra type is not rate-limited across types**: a space listing `report.filed` gets at most one a minute, all types counted together. **Recommendation: accept**, as the plan's "one per space".
