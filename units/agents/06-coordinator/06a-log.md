---
plan: agents/06-coordinator (06a)
job: agents-06a
date: 2026-10-24
---

# Log: agents/06a, the mechanism: leases, `wsm listen`, cursors

## What was built

**core** (rebased onto main after 04a merged; events.sh's type list merged by hand):

- `lib/anfs/files.sh` (bootstrap set, parses under bash 3.2):
  - `anfs_lock <dir> [tries]` and `anfs_unlock`: report.sh's mkdir lock, moved and shared. The pid inside (`BASHPID`). A dead pid's lock is broken under a `<lock>.break` guard, and only if it still holds that pid, so two breakers never remove a lock just taken. A directory that cannot be written fails at once, not after 10 s of retries.
  - `anfs_write_atomic <file> <text> [mode]`: temp file in the same directory, mode, fsync, rename. Linux: `chmod`, `sync -- FILE`, `mv`. macOS: one perl does chmod, `IO::Handle->sync` and rename (three forks became one; see the CPU note).
  - report.sh's render uses the lock.
- `lib/anfs/events.sh`:
  - Cursors: a byte offset per month file, one JSON line, `{"v":1,"files":{"<scope>/<YYYY-MM>[.bus].jsonl":N},"last":"<id>","at":"<UTC>"}` (`claim` too for a pending one).
  - `anfs_events_cursor_init` (the ends of the newest month's files), `_read`, `_save` (atomic, 0600).
  - `anfs_events_from <cursor> <fn>`: skips to each offset with `LC_ALL=C read -N` on the descriptor, hands the backlog in log order, keeps the newest month's descriptors for following. `anfs_events_follow_from` adds the follow.
  - `_anfs_ev_drain` tracks positions (`_EV_POS`) when `_EV_TRACK` is on.
  - `anfs_events_follow <fn> [deadline] [tick fn]`: the tick fn runs after each tick's reading; non-zero returns 2.
  - `events ls --cursor FILE` (wsm's and anfs's).
  - `--after`'s help: "by id order, not log position ... to resume without a gap, use --cursor".
  - `lease.claimed lease.released lease.expired` in `ANFS_EVENT_TYPES`.
- `lib/anfs/lease.sh` (new):
  - The lease file, its parse and write (one line, fixed layout, read with bash regexes, no jq).
  - Liveness: pid with `ps -o lstart=` squeezed, and TTL by state.
  - `anfs_lease_claim`, `_renew`, `_release`, `_listen_start` (the pending cursor committed or discarded, the listener registered), `_listen_end`.
  - The time function `_anfs_lease_now` (`ANFS_TEST=1` + `ANFS_LEASE_NOW`, a number or `@file`). Epoch from ISO in pure bash, with no `date` (BSD/GNU differ).
- `wsm` 0.13.0 (package 0.1.13 by the hook):
  - `wsm lease list|holder|claim|renew|release`.
  - `wsm listen`.
  - README rows.
- The wsm test helper:
  - Every `XDG_*` (including `XDG_RUNTIME_DIR`), `ANFS_{CONFIG,DATA,CACHE,STATE}_HOME` and `WSM_*_HOME` are set into the test's temp dir.
  - `ANFS_RUN`, `ANFS_EVENTS_SPOOL`, `ANFS_ACTOR`, `ANFS_AS`, `ANFS_SPACE`, `AI_AGENT`, `CLAUDECODE`, `CLAUDE_*` are unset.
  - **`wsm_guard`** resolves the directories through `paths.sh` and fails the test when any is outside its temp dir or under the real home.
- AGENTS.md: a "Leases" section, and `--cursor`/`--after` under "Events".

**events** (v1.1.0, not tagged):

- Commit 1, 04a's types, as the coordinator asked:
  - `run.auth-required` and the new `run.*` keys.
  - Approvals (six types) and Inbox (`inbox.dismissed`).
  - `source.updated` by `anfsd`.
  - Examples `run-auth-required`, `approval-requested`, `approval-given`.
  - SPEC.md and the manifest at 1.1.0, and CHANGELOG.
- Commit 2: "Leases (core: `wsm lease`)", with examples `lease-claimed`, `lease-expired`, `lease-released`.
- Every example was written by core's real `anfs_emit` (into a scratch spool), then a few ids in data were edited to plausible times.
- The schema and its `$id` (v1.0.0) are unchanged: no rule changed.

**docs**: this log.

## Commits (branch agents-06a, not pushed, not tagged)

| Repo | Commit | What |
| --- | --- | --- |
| core | `56a3a52` | files.sh: `anfs_lock`, `anfs_write_atomic`; report.sh uses the lock; tests |
| core | `1a0278c` | events.sh: cursors, `events ls --cursor`, the tick fn, `--after` help, `lease.*` types; tests; settings.bats knows `ANFS_TEST`/`ANFS_LEASE_NOW` |
| core | `7796219` | lease.sh, `wsm lease`, `wsm listen`, the helper's isolation and guard, lease.bats, listen.bats, README; wsm 0.13.0 |
| core | `d1eead9` | AGENTS.md |
| events | `1528b74` | 04a's types, 1.1.0 |
| events | `4de70eb` | `lease.*` |
| docs | (this) | this log |

Versions:
- anfs stays 0.2.19: 04a already bumped it in the unpushed series, so the hook did not bump it again. **If main is pushed before this branch is rebased, bump anfs by hand.**
- core/wsm goes to 0.1.13 (the hook), and `WSM_VERSION` to 0.13.0 (by hand).

## Decisions made during the work (deviations from the plan, and why)

1. **Cursor keys are `<scope>/<file>`** (`acmeroot22/2026-10.jsonl`), not the bare `2026-10.jsonl` of the plan's example. `events ls --cursor` reads several scopes (`anfs events ls` reads all of them), so one shape serves one scope and many. Entries of scopes a reader does not read are kept as they are.
2. **The rule for files not in the map:** a file is read from 0 when its month is at least the newest month the map has for its scope (or, when the map has none for the scope, the month of `at`). An older one is done. A cursor that does not parse is warned about and read from the start of the newest month (a duplicate, never a gap).
3. **`events ls --cursor` prints in log order**, file by file, not merged by id. Without `-f` it saves once at the end. With `-f` it also saves after each printed line, so `-f --once --cursor F` takes one event per run.
4. **The column is `STATUS`, not `STATE`** (`wsm lease ls`, `holder`, `--json` key `status`). Core's registry test refuses `STATE` in any list (04a hit the same). `holder` prints `SPACE ROLE HOLDER SESSION PID STATUS SINCE RENEWED`.
5. **`release` with a lost claim exits 0** and leaves the file unchanged, as the plan's help text says. The plan's test list said "1" for both renew and release. `renew` with a lost claim is 1.
6. **`release --reason explicit|session-end|clear`** was added; 06b's hook needs it. The default is `explicit`.
7. **The listener's start time is checked at each renewal**, not each second: `ps` is a fork, and `kill -0` (no fork) runs every tick. A pid reused within 30 s is caught at the next renewal.
8. **Lines that do not wake are not committed until the next delivery.** The listener writes the pending cursor only when it delivers a batch, and on `--max`. Each `listen` therefore rereads the lines since the last delivery that did not wake it. This is cheap, and it means a holder that never runs `listen` again loses nothing. Exit 1 (lost, dead) writes nothing.
9. **The default wake set is a constant** (`ANFS_LEASE_WAKE`); `coordinator: wake` in space.yml is 06b's. 04a's types now exist, so `run.auth-required` and `approval.requested` already wake.
10. **The batch's "more" line** is printed only past 50 events: `... N more: wsm events ls --after <50th id>`. It is informational (the cursor already moved past them). There is no `inbox:` pointer; that is 06b's prompt line. "since" is the cursor's `at` (the last commit), else the listener's start.
11. **Renewal failures** are logged as `warn` once when they start and once on recovery, and each attempt as `debug` (the tests count those). With `--hook`, they go to the log only, because stderr is the batch's channel.
12. **The idle CPU test measures the difference between a 14 s run and a 4 s run.** It uses the wrapper's `times` (children included: ps, perl), so start and end are left out. Renewal is every 5 s (`--ttl 15s`), which is denser than the real 30 s.
    - On macOS a renewal first cost about 50 ms of CPU (mkdir, ps, chmod, perl, mv, rm). Doing chmod, fsync and rename in one perl brought it to about 30 ms.
    - Results: about 110–135 ms over 10 s with two renewals, and about 20–40 ms while every renewal fails. The limit is 200 ms.
    - With a renewal every second it would be about 285 ms. That is not a real setting.
13. **The listener ends itself on SIGTERM/HUP/INT** (an `asyncRewake` timeout, ^C). It sets the lease back to `working` when the lease is still its own, writes no pending cursor, and exits 143/130.
14. **`lease.claimed` is written after the cursor is made**, so a first claim's cursor ends just before its own event. That event does not wake (not a wake type, and it is the holder's own).

## Tests

All through `anfs work run agents-06a <repo> -- ... </dev/null`.
- **New bats:** `wsm/tests/lease.bats` **16**, `wsm/tests/listen.bats` **15**; `anfs/tests/events.bats` 32 (+5, `--cursor`); `anfs/tests/files.bats` 9 (+2, the lock and the atomic write).
- **Affected suites:** all of `packages/wsm/tests` with `events.bats`, **185 passed**; `report.bats`, `bash.bats`, `registry.bats`, `settings.bats` pass.
- **`packages/anfs/tests/all` (pmm and pcm off):** **811 passed, 0 failed**, 58 files, 123 s.
  - A first run had 2 failures, both fixed: the `STATE` column (deviation 4) and the test-only clock variables in `settings.bats`.
- **What the listener tests cover:**
  - the batch (stdout, exit 0; `--hook` stderr, exit 2);
  - the window and the 50 lines plus the count;
  - `--type` and self-exclusion;
  - **a late line with an older id delivered where `--after` skips it**, and a cnfs `.bus.jsonl` line;
  - a new month's file mid-wait;
  - pending committed by the same claim, and redelivered to a new holder;
  - atomic renewal: a reader looping for 4 s never sees the file missing or partial;
  - **back-off on the fake clock**: attempts at +30, +60, +120, +240, +540, a single failure line and a single recovery line, still listening;
  - holder killed → exit 1 within 2 s;
  - a newer listener → the older exits 1;
  - released → exit 1;
  - `--max` → 3;
  - the two CPU bounds.
- **What the lease tests cover:**
  - the first claim (event, cursor, modes 0700/0600);
  - 75 with no event;
  - a re-claim renews;
  - takeovers: dead, reused, ttl;
  - `working`'s TTL, `listening`'s TTL;
  - lost claims;
  - renew `--working`;
  - release and `from=released`;
  - 77 under `ANFS_RUN`;
  - usage errors;
  - **20 parallel claims → one claim, one event**;
  - a stale lock broken;
  - `ls`/`holder` read only.
- **events:** `tools/check` (check-jsonschema 0.38.2 from `tools/requirements.txt`, venv in the job's TMPDIR): `70 examples (26 valid, 44 invalid): ok`. The 04a commit alone: 67, ok.
- **Isolation:**
  - The new suites run with PATH = the system's directories plus the directories of bash and yq.
  - `events.bats` uses `env -i`.
  - No test reads or writes the real `~/.local/state`, `~/.config` or `~/.local/share`.
  - Manual checks ran in a scratch HOME under the job's TMPDIR, with `env -i`.

## Not run

- Linux. The code avoids BSD/GNU differences (no `date`, no `stat` outside test fallbacks, `sync -- FILE` with a perl fallback, `ps -o lstart=` under `LC_ALL=C`), but no Linux run was made.
- `anfs events validate` over the `lease.*` lines: that needs anfsd. The examples come from core's emitter and pass `tools/check`.
- Nothing installed. No `wsm lease`/`listen` against real state; no bus, anfsd, ui or Claude.

## By hand, for the person (no migration code)

1. Merge: rebase `agents-06a` in core onto main if main has moved; ff-merge core, events and docs. If core's main was pushed with 04a's anfs 0.2.19 before this merge, bump `packages/anfs/package.yml` by hand.
2. `anfs src update core`; `ppm install core/anfs core/wsm`.
3. In anfs-dev, `wsm lease ls`: nothing yet (no lease files).
4. Optional:
   1. `wsm lease claim coordinator --kind shell` in a terminal (the pid is that shell's).
   2. `wsm listen --claim <id> coordinator`.
   3. File a test report from another space; see the batch.
   4. `wsm lease release --claim <id> coordinator`.
5. events: read the two commits and tag `v1.1.0` when content. Then copy the tagged `examples/` into core's `packages/anfs/tests/fixtures/events-spec/` and runtime's `testdata/spec/` with a new `VERSION`, as 05b says.

## Open questions

1. **Should `wsm listen` commit the cursor when it ends without delivering (exit 1 on a lost lease)?** Today it does not (deviation 8), so the next holder rereads from the last delivery. **Recommendation: keep it.** Rereading non-waking lines is cheap, and committing on loss would race the new holder's cursor.
2. **Renewal cost on macOS:** about 30 ms of CPU per renewal, mostly perl's start for the fsync. **Recommendation: accept it** (one per 30 s per listening role). If it ever matters, a tiny `anfs-fsync` helper in Go (runtime ships binaries already) could replace perl.
3. **The events schema's `$id` still names v1.0.0.** The schema did not change, so v1.1.0 vendors the same file. **Recommendation: leave it**, and bump `$id` only when the schema changes.
4. **ppm's tests leave `ppm-messages.*` and `ppm-skip.*` files in TMPDIR** (seen in the job's `/tmp/anfs-work-501-3`). Not this job's code. **Recommendation: a small fix in ppm's helpers** (clean up in teardown).
