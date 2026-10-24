# Plan 06 — a coordinator per space: `wsm lease`, `wsm listen`, Claude hooks, notifications

Written 2026-10-24 in the job `agents-06-plan` (docs only; code read, nothing run but `claude --version`, `ps` on this session's own ancestry and `strings` on the installed claude binary). Against core `a90652c`, runtime `367ee0c`, events (local, untagged), Claude Code 2.1.296.

Revised 2026-10-24 after Claude Desktop's review (agreed by the person and the coordinator): the cursor is a byte offset per file, not id order, with `--after` documented as id order (section 3); renewals are atomic and back off on failure (section 1); what a fresh renewal does and does not mean (section 1).

Binding: `AGENT-NATIVE.md` (principle 9 "anfs ships mechanisms; workflows are tenants", identity `<space>/<role>`, actor and role as separate event fields, durable before real time, plain files, the bus guards itself), `AGENT-NATIVE-REVIEW.md` §4.1 and §4.3, `units/agents.md`, plan 04 (04a is being built now; 06 depends on it), 02d's plan and log (a dispatched claude runs with `--setting-sources=`: no user, project or local file, so no project hooks), `reviews/claude-settings.md` ("Decided 2026-10-24": integrations are owned by the package that installs them; no hooks in source repos without review).

**Depends on 04a** (`approval.*`, the inbox records, `anfs inbox`). 06a (the mechanism) does not, and can start now.

**For the person's review.** Each section is a decision: the recommendation first, the alternatives in a line. The open questions are at the end, each with a pick.

## Context — read these files first

The jobs would be `anfs work new agents-06a core events docs`, `anfs work new agents-06b core runtime anfs-dev docs`, `anfs work new agents-06c runtime core anfs-dev docs`.

- docs: `AGENT-NATIVE.md`, `AGENT-NATIVE-REVIEW.md` §4, this plan, `units/agents/04-inbox/plan.md` (sections 1, 3, 3a, 5), `units/agents/02-daemon/02d-safe.md` section 3 and `02d-log.md` (`--setting-sources=`, the per-run spool and its late ingestion), `units/agents/05-spec/05b-log.md` (the recorder's `.bus.jsonl`), `reviews/claude-settings.md`
- core: `packages/anfs/home/.local/lib/anfs/events.sh` (`anfs_events_space_at`, `_anfs_ev_actor`, `anfs_events_ls`, `anfs_events_follow_init`, `anfs_events_follow`, `_anfs_ev_drain`), `lib/anfs/report.sh` (`_anfs_report_lock`: the mkdir lock this plan reuses), `lib/anfs/paths.sh` (`ANFS_STATE_HOME`), `packages/wsm/home/.local/bin/wsm` (`_meta_load`, `_mf_plan`, `_home_ok`, install and remove), `packages/anfsd/home/.local/bin/anfs-agents` (the CLI shape), AGENTS.md ("Events", "`space.yml`")
- runtime: `go/internal/supervise/supervise.go` (`alive`, `psStart`: the pid plus `ps -o lstart=` check this plan copies), `go/internal/space` (`dispatcher: anfsd|none`, the precedent for `coordinator: claude|none`), `go/internal/sink` (a consumer's shape, for 06c)
- events: `REGISTRY.md` (anfs's catalog), `examples/`, `tools/check`
- Claude Code: https://code.claude.com/docs/en/hooks.md and settings.md (facts below, "Sources checked")

## Overview

Today the anfs-dev coordinator session keeps itself awake by hand: it runs `anfs events ls -f ... | grep -m1 ...` as a background task, the task's exit wakes the idle session, and the model must remember to start it again. 06 turns that into a mechanism any space can opt into:

1. **A lease per role and space** (`wsm lease`, also `anfs space lease`): `claim`, `renew`, `release`, `holder`, `list`. Any tool can hold a role; Claude Code is the first, Firstmate or herdr later. A lease has a TTL and is renewed; a lease past its TTL, or held by a dead process, can be taken.
2. **A listener** (`wsm listen`): resumes from the lease's cursor, waits for the space's wake events, gathers a short batch, and exits so whatever runs it wakes up. While it waits it renews the lease.
3. **Events** `lease.claimed`, `lease.released`, `lease.expired`, so the ui and anyone else see who holds a role.
4. **Policy in `space.yml`**: `coordinator: claude|none` (and the wake set); **wsm writes the space's `.claude/settings.json`** with the hooks that make the first Claude session in the space its coordinator: the first piece of settings ownership.
5. **The Claude adapter** (one hook script, shipped by core/anfs): SessionStart claims or reports the holder; every prompt checks the holder and takes over a dead or expired lease; Stop and SessionStart arm the listener as an `asyncRewake` hook, so a wake never depends on the model rearming; SessionEnd releases.
6. **Independent of the lease**: a prompt hook in every session of a managed space shows pending inbox items, and anfsd sends OS notifications on `approval.requested`.

**Framing** (the person's): the interactive coordinator is the pleasant "an agent keeps me company in this space" experience, not where reliability lives. Unattended triage belongs to a dispatched `<space>/coordinator` role (02b's dispatch); approvals need the person, so the prompt hook and the notifications do most of that job. Nothing in 04's guarantees rests on a session being open.

Done when: in anfs-dev, the person starts Claude Code at the space's root; it says it is the coordinator; a report filed from cnfs wakes it within seconds, with a batch of the events; a second session in the space says who the coordinator is and does not listen; closing the first and typing in the second makes the second the coordinator, resuming after the last event the first saw; a dispatched run never holds the lease; `anfs inbox` items show as one line in every session's next prompt; an approval request pops an OS notification on macOS and on Linux.

## 1. The lease (mechanism)

**Recommendation.** A lease is a small JSON file per space id and role in STATE, written under a lock, owned by core's bash library (`lib/anfs/lease.sh`), used by `wsm lease` and `wsm listen`. Nothing in it is Claude's: Claude is one kind of holder.

**Where:** `$ANFS_STATE_HOME/leases/<space id>/<role>.json` (`~/.local/state/anfs/leases/tkj76y3nfx/coordinator.json`), beside it `<role>.cursor` (section 3) and the lock `<role>.lock/` (a mkdir lock with the holder's pid inside, stale when that pid is dead: `report.sh`'s `_anfs_report_lock`, moved into `lib/anfs/files.sh` and shared). By space id, like the event log, so a lease survives `wsm move`. Directories 0700, files 0600. **Every write is atomic**: a temp file in the same directory, fsync, rename (the shipper's discipline for `cursors.json`), through one helper `anfs_write_atomic` in `files.sh`. bash has no fsync, so the helper calls `sync -- FILE` where coreutils has the file form (Linux) and `perl -MIO::Handle -e 'open(my $f, "<", $ARGV[0]) or exit 1; $f->sync or exit 1' FILE` where it does not (macOS, whose `sync` takes no file); both are base software on the two platforms.

**Format** (`v: 1`):

```json
{
  "v": 1,
  "space": "tkj76y3nfx",
  "spacepath": "anfs-dev",
  "role": "coordinator",
  "claim": "019a1f3c-5d2e-7b41-9c3a-0f4e2d1b7a66",
  "holder": {
    "actor": "claude:5a52e1c0",
    "kind": "claude",
    "session": "5a52e1c0-1b2c-4d5e-8f90-123456789abc",
    "pid": 21363,
    "pstart": "Wed Oct 21 15:26:45 2026"
  },
  "state": "listening",
  "claimed": "2026-10-24T10:42:07Z",
  "renewed": "2026-10-24T11:03:37Z",
  "ttl": 90,
  "work_ttl": 7200,
  "listener": {"pid": 48211, "pstart": "Sat Oct 24 10:58:02 2026"}
}
```

- `claim` is a UUIDv7 made at each claim: the fencing token. Every `renew`, `release` and listener write checks that `claim` is still its own, so a holder that lost the lease can never write over the new one.
- `holder.session` is an opaque string the claimant gives (Claude's `session_id`; a pane id for herdr). `holder.pid` is the process whose life is the lease's life; `pstart` is `ps -o lstart= -p <pid>` with its whitespace squeezed, exactly anfsd's `psStart` (same on macOS and procps), so a reused pid is not the holder.
- `state`: `listening` (a listener is up and renews every 30 s) or `working` (claimed, or woken and busy: no listener for now). Two TTLs because a woken session works for minutes or hours before it waits again: `ttl` (default 90 s) applies while `listening`, `work_ttl` (default 2 h) while `working`.

**Live, or takeable.** A lease is **live** when all hold: the file parses; `holder.pid` exists (`kill -0`, EPERM counts as alive) with the recorded `pstart`; and `renewed` is within `ttl` (`listening`) or `work_ttl` (`working`). Otherwise it is takeable: `free` (no file, or released), `dead` (the pid is gone or reused), or `expired` (stale renewal, **even when the pid lives**: the person's "a real lease"). `released` leases keep their file (and the cursor) with `holder` removed and `state: free`.

**Heartbeat: the running listener renews, it does not exit to be rearmed.** While it waits, `wsm listen` rewrites `renewed` every 30 s (a third of `ttl`) and checks its holder pid each second (it already ticks once a second to follow the log). It exits only for real events, when its holder dies, or when the lease is no longer its own. *Why, against Claude Desktop's "exit every N minutes, rearm me":* each exit wakes the session and costs context and a model turn; a renewal is a 300-byte atomic write. **A failed renewal backs off** (the no-spin rule): the listener retries at 30 s, 60 s, 120 s, then every 5 minutes, logs the first failure and the recovery once each (`anfs log`), and keeps listening; it never retries in a loop. If the failures outlast `ttl` the lease simply expires and another session may take it, which is the right outcome for a holder that cannot write its state. With a renewing listener, "live" means the listener is alive and the holder is alive, and the session is woken only for events. The one exception is a cap imposed by the harness (section 5: a hook's timeout), handled there as a rare rearm, not as the heartbeat.

**What a fresh renewal means, and does not.** It means the listener process is alive and its holder process is alive. It does **not** mean the session acts on what it hears: a woken session may ignore the batch, misread it, or be busy with the person. That is acceptable by design: the interactive coordinator is a companion, not where reliability lives. The human side is covered by the inbox line in every session's prompt (section 7) and the OS notifications (section 8); the work is covered by dispatched runs (02b) and 04's inbox for what no run took. Nothing in 06 or 04 waits on a coordinator having acted.

**Claims are refused in a dispatched run.** `wsm lease claim` exits 77 when `ANFS_RUN` is set (anfsd sets it in every run): a run is a task, not a resident; leases are for long-lived holders. This is the second guard; the first is that a dispatched claude loads no project hooks (section 5).

*Alternatives:* liveness by pid alone (a hung or deaf session holds the role forever); a listener that exits every N minutes (a wake per N minutes per idle session); the lease in NATS KV (anfsd and NATS become required for something files do; principle 2); `flock` (not on macOS without a package).

## 2. `wsm lease` and `wsm listen`: the commands

`anfs space` is wsm (anfs's tool alias), so these are wsm commands and `anfs space lease ...` works as the person wrote it. The space is the one at `.` (`anfs_events_space_at`), or a SPEC where it says so. `--json` on every command.

```
$ wsm lease --help
Usage: wsm lease <command> [args]

Commands:
  list, ls [spec]                   Every lease of the spaces under SPEC (none: every space): SPACE ROLE HOLDER
                                    STATE SINCE RENEWED. STATE is listening, working, expired, dead or free
  holder [--space SPEC] <role>      Print the role's holder in the space at . (its actor, session, pid and state);
                                    exit 0 when the lease is live, 1 when it is free, expired or dead
  claim [options] <role>            Take the role in the space at . when it is free, released, expired or held by
                                    a dead process; print the claim id. A live lease held by another is not taken
                                    (exit 75, its holder on stderr). Claiming again as the same holder renews it
                                    --session ID   the holder's session (default: the pid's)
                                    --pid PID      the process whose life is the lease's (default: the caller's parent)
                                    --kind KIND    what holds it: claude, herdr, firstmate, shell (default: shell)
                                    --ttl 90s --work-ttl 2h
  renew [--working] --claim ID <role>
                                    Mark the lease alive now (--working: busy, no listener; the longer TTL);
                                    exit 1 when the claim is no longer the holder's
  release --claim ID <role>         Give the role up; the cursor stays for the next holder. Exit 0 also when the
                                    claim had already lost the lease

A lease is a role held in a space by one process at a time: $ANFS_STATE_HOME/leases/<space id>/<role>.json.
It is live while its process runs (the pid with the start time it was claimed with) and its renewal is
younger than its TTL; anyone may take a lease that is not, even from a process that still runs.
Claims, releases and expiries are events (lease.claimed, lease.released, lease.expired). A dispatched run
cannot claim (exit 77).
```

```
$ wsm listen --help
Usage: wsm listen [options] --claim ID <role>

Wait for the events that wake ROLE's holder in the space at ., then print them and exit 0. Resumes after
the last event the role's holder was given (the lease's cursor: byte positions in the log, not event
ids; from now at the very first claim), so
nothing that happened between two holders is missed. Renews the lease every 30s while it waits.
After the first event, it waits WINDOW for more and prints them together (at most 50 lines, then a count
and the command that lists the rest).

  --type GLOB      Wake on these types (repeatable; default: the space's coordinator: wake list)
  --window 5s      Gather the batch for this long after the first event
  --max DURATION   Give up after this long: exit 3 with nothing printed (default: none)
  --hook           Claude Code asyncRewake form: the batch on stderr and exit 2; nothing on stdout

Exit: 0 events printed; 1 the claim is no longer the holder's, or the holder process ended (nothing printed);
2 the --hook form's events; 3 --max passed. Events by the holder itself never wake it.
```

`wsm lease ls` output, for the record:

```
SPACE     ROLE         HOLDER            STATE      SINCE  RENEWED
anfs-dev  coordinator  claude:5a52e1c0   listening  10:42  12s ago
cnfs      coordinator  -                 free       -      -
```

Exit codes follow the toolkit's: 64 usage, 75 (EX_TEMPFAIL) held by another, 77 (EX_NOPERM) in a run, 1 the documented "no".

**What the listener prints** (the batch Claude reads), one line per event, then where it resumes:

```
anfs-dev: 3 events since 10:58:02Z (coordinator)
  2026-10-24T11:04:10Z report.filed       r-20261024-3   cnfs/partitia  claude:91be03aa  core/pcm: compose up ignores x-pcm.depends_on
  2026-10-24T11:04:12Z run.submitted      019a1f…7c2d    anfs-dev       anfsd            rule fix-core-report
  2026-10-24T11:04:13Z approval.requested 019a1f…88e1    anfs-dev       anfsd            merge core +3 (r-20261024-2)
more: wsm events ls --after 019a1f3c-...  |  inbox: anfs inbox
```

Data is shown for the types where one key says what happened (`title` for reports, `rule` for runs, 04's WHAT for approvals); everything else is the row `events ls` prints.

*Alternatives:* `anfs lease` as a new top-level group (a lease is a space's, and `wsm events` already reads "the space you are in"); listen as an `events ls` flag (it would need to know about leases and renewals: two concerns in one command).

## 3. The cursor: no gap between holders

**Recommendation: a log position per file (byte offsets, like the shipper's `cursors.json`), kept beside the lease** (`<role>.cursor`), with the last event id for display only. Not id order, and not a NATS durable consumer.

What the code shows: `events ls --after ID` compares ids as strings (`events.sh`'s `_anfs_ev_match`: `[[ "$EV_ID" > "$_EVF_AFTER" ]]`). Ids are UUIDv7, so that is time order **of creation, not of arrival in the log**; several writers and clock skew between them make it worse. Two kinds of line arrive late with an older id: a dispatched run's own events, which anfsd ingests from `runs/<id>/events/` a second or more after they were written (02d; at anfsd's next start when it was down), and the recorder's `.bus.jsonl` lines for events another layer published straight to NATS (05b). An event created at 10:01 can be appended after one created at 10:02; a listener resuming `--after <last id>` would skip it silently. The month files are append-only (one `O_APPEND` write per line, never rewritten), so a byte offset per file is exact.

**Format** (`v: 1`):

```json
{"v": 1, "files": {"2026-10.jsonl": 418233, "2026-10.bus.jsonl": 2071}, "last": "019a1f3c-5d2e-7b41-9c3a-0f4e2d1b7a66", "at": "2026-10-24T11:04:13Z"}
```

- `files`: per month file of the space's scope directory, the byte offset just past the last complete line consumed (`{"<YYYY-MM>.jsonl": <offset>}`, the shipper's shape, relative to the scope). A partial last line is never consumed. Files of older months not in the map are done; a newer month's file not in the map starts at 0. `last` is the last delivered event's id, for people and for the "more:" line, never for resuming.
- **What events.sh gains (06a):** `anfs_events_follow_from <cursor file>`, which opens each month file the cursor names, skips to its offset on the descriptor (`LC_ALL=C read -r -N <offset>`, no seek in bash; month files are at most a few MB), reads the backlog to the end, and then follows exactly as `anfs_events_follow` does; and `anfs_events_cursor_save`, which writes the offsets reached (each line's length plus its newline, counted under `LC_ALL=C`) through `anfs_write_atomic`. An offset beyond a file's size (a file that should never shrink did) is warned about once and that file is read from 0: a duplicate, never a gap.
- At a month change the old month's files are read to their end from their positions, then the new month's from their start: what `anfs_events_follow` already does.
- **At-least-once, committed by the next arm.** When `wsm listen` delivers a batch it writes `<role>.cursor.pending` (the positions after the batch, and the claim id). The next `wsm listen` with the **same claim** promotes pending to the cursor before it waits: the session that was woken has had its turn. A different claim (a new holder: the old one died before acting) discards pending and starts from the committed cursor, so the batch is delivered again. A duplicate costs a line of context; a gap costs a missed report.
- **From now only at the very first claim** of a role in a space (no cursor file): the cursor is set to the ends of the current month's files. `release` keeps the cursor; the next holder resumes from it.
- **The same mechanism for everyone**: `events ls [-f] --cursor FILE` (06a) reads from the positions in FILE (from the ends of the current files when FILE does not exist yet) and saves the positions reached after each printed line, so any tool (Firstmate, a script) can resume exactly. `wsm listen` uses the library functions, with the pending step.
- `--after`'s help says what it is: "--after ID: Only events after this event id, by id order, not log position: a line can arrive late with an older id (a run's own events, another layer's); to resume without a gap, use --cursor".

**A NATS durable consumer** (`coordinator-<space id>` on `anfs.<space id>.>`) was weighed: it orders by arrival and acks per message, which is what we want; but it makes the coordinator deaf whenever anfsd or NATS is down (the files still have everything), needs a NATS client and credentials in the hook path, leaves a consumer behind for every space ever coordinated, and duplicates what the files already give. Principle 1 and 2 favour the files. It stays the right answer for a tool on another host, later.

*Alternatives:* the last-seen id in the lease (simple, but loses late lines, as above); an id plus a look-back window and a seen set (works, but two parameters to get wrong and a set to keep).

## 4. Events

Three types, in core's `ANFS_EVENT_TYPES` and in anfs-io/events' REGISTRY.md with examples (a new "Leases (core: `wsm lease`)" table; the examples' manifest and `tools/check` updated). Scope: the lease's space. Subject: the role. Actor: the claimant (`_anfs_ev_actor`: `claude:<session8>` in a Claude session; `ANFS_ACTOR` when a tool sets it). `actorrole` is the role on `lease.claimed` and `lease.released` (the actor acts as it from then, or did until then).

| Type | When | Data |
| --- | --- | --- |
| `lease.claimed` | a claim took a free, released, expired or dead lease (not a re-claim by the same holder) | `claim`, `kind`, `session`, `pid`, `from` (`free`, `released`, `expired`, `dead`), `previous` (the previous holder's actor, when there was one) |
| `lease.released` | the holder gave it up | `claim`, `reason` (`session-end`, `clear`, `explicit`) |
| `lease.expired` | written **by the taker**, just before its `lease.claimed`, when it takes an expired or dead lease | `claim` (the old one), `holder` (the old actor), `why` (`ttl`, `dead`, `reused`), `renewed` (the last renewal) |

No `lease.renewed`: a renewal every 30 s per space is noise on the bus, and principle 7 (a client's reads emit nothing) extends to keep-alives. Nobody watches for expiry by polling: it is noticed and recorded by whoever takes the lease. The ui keeps the holder from these three events (06c).

## 5. The Claude adapter: hooks

**Facts checked** (hooks.md and settings.md, fetched 2026-10-24; installed 2.1.296):

- Every hook gets JSON on stdin with `session_id`, `transcript_path`, `cwd`, `permission_mode`, `hook_event_name` (and `agent_id`/`agent_type` inside a subagent). SessionStart adds `source` (`startup`, `resume`, `clear`, `compact`, `fork`), SessionEnd `reason` (`clear`, `resume`, `logout`, `prompt_input_exit`, `other`), UserPromptSubmit `prompt`.
- **SessionStart** and **UserPromptSubmit**: plain stdout and `hookSpecificOutput.additionalContext` are added to Claude's context (UserPromptSubmit's each as a system reminder starting with the hook's name). UserPromptSubmit's default timeout is 30 s; on timeout its output is dropped and the prompt goes through. SessionStart hooks can append `export` lines to `$CLAUDE_ENV_FILE` for the session's later Bash commands.
- **SessionEnd** cannot block; its hooks share a 1.5 s budget by default (raised by a per-hook `timeout`, up to 60 s). It does not run when the process is killed: the dead-pid rule covers that.
- **`async: true`** runs a command hook in the background; its output is delivered on the next turn, and "if the session is idle, the response waits until the next user interaction. **Exception: an `asyncRewake` hook that exits with code 2 wakes Claude immediately even when the session is idle**", its stderr (or stdout) shown to Claude as a system reminder. The timeout is **not** enforced on plain async hooks but **is** on `asyncRewake` ones. In `-p` mode async hooks still running at teardown are killed; for interactive sessions the page says nothing, so the listener must end itself when its holder dies.
- Hooks run in parallel when several match one event; "in their own session without a controlling terminal"; spawned by the Claude Code process, `sh -c` for the shell form, directly for the exec form (`args`). They inherit Claude's environment (2.1.296 also exports `CLAUDECODE=1`, `CLAUDE_CODE_SESSION_ID` and an undocumented `CLAUDE_PID` to its children, seen in this session's own environment).
- **Project settings are read from the session's primary working directory only** (`.claude/settings.json` "from the session's primary working directory, so to use a file committed at the repository root, start Claude Code there"). Hooks from any settings file wait for the **workspace trust** dialog in an interactive session; `-p` treats the folder as trusted. Claude Code writes standing approvals to `.claude/settings.local.json`, never to `settings.json` from the CLI.

**Recommendation: one script, five entries, the listener armed by hooks, not by the model.**

The script is `~/.local/lib/anfs/claude/hook` (core/anfs's package, beside the libraries it sources: `paths.sh`, `events.sh`, `lease.sh`; no wsm startup, no yq on the prompt path). It reads the hook's JSON with the bash parsing `events.sh` already uses (no jq). Its first argument is the entry:

| Hook | Entry | Does |
| --- | --- | --- |
| SessionStart (sync, timeout 10) | `session-start` | resolve the space from `cwd`; verify the Claude pid; claim (or re-claim on `compact`/`resume`, same session); print the coordinator brief, or one line naming the live holder |
| SessionStart (`async`, `asyncRewake`, timeout T) | `listen` | if this session holds the lease, run the listener (`wsm listen --hook`); else exit 0 at once |
| Stop (`async`, `asyncRewake`, timeout T) | `listen` | the same: rearms at the end of every turn, including the turn a wake started |
| UserPromptSubmit (sync, timeout 5) | `prompt` | the takeover check, then the inbox line (section 6); silent when there is nothing to say |
| SessionEnd (sync, timeout 3) | `session-end` | release when this session holds it (`reason` → `session-end` or `clear`); never blocks |

**The prompt check**, on every prompt (budget: one file read and a `ps` in the common case):

1. Not in a space, `ANFS_COORDINATOR=off`, `ANFS_RUN` set, or no verified Claude pid: skip to the inbox line.
2. Read the lease. Held by this session and `claim` is ours: nothing to say (the Stop hook rearms at this turn's end). Held by this session's pid but we lost it (another claim): say once "this session is no longer the coordinator of anfs-dev (claude:91be03aa took it at 11:20)".
3. Held live by another: nothing (SessionStart already said who).
4. Takeable (free, released, expired, dead): claim; on success print the brief ("this session is now the coordinator of anfs-dev, taken from claude:5a52e1c0 (dead)"); the listener starts at this turn's Stop.

**The brief** (stdout of `session-start` or `prompt` when it claims; about 6 lines): you are `<spacepath>/coordinator` (id); events of the types in `coordinator: wake` wake you with a batch; act on what is yours, tell the person what is theirs (approvals: `anfs inbox`); `wsm events ls --after <id>` for more; `wsm lease holder coordinator` to check; other sessions in this space do not listen; `ANFS_COORDINATOR=off claude` starts a session that never coordinates. On `compact` the brief is printed again (the old context is gone).

**Why `asyncRewake` hooks and not a background task the model starts.** Today's way (the model runs the listener with Bash `run_in_background`; the task's exit wakes the session) needs the model to start it, and to start it again after every wake. A Stop hook fires at the end of every turn, wakes included, so the rearm happens without the model; the SessionStart one covers the time before the first prompt. Several `listen` entries can run for one session (one per turn); each writes itself into the lease's `listener` (newest wins, under the lock) and an older one that sees another pid there exits 1 at its next tick (not 2: no wake). The SessionStart pair runs in parallel: `listen` claims too when the lease is free (claim is idempotent for the same holder), so the order does not matter.

**The timeout cap.** `asyncRewake` hooks have their timeout enforced. wsm writes `timeout: T` with T = 86400 s (a day) and the hook runs `wsm listen --hook --max (T - 60 s)`; when listen exits 3 (`--max`), the hook exits 2 with one line ("anfs: no events in 24h; still listening") so the turn it starts rearms it: one small wake per idle day. **06b's first task checks T**: the largest `timeout` 2.1.296 accepts on an `asyncRewake` hook, from the binary and a fake hook in a scratch project (the person's rule allows `claude` there; not in a space). If it is under an hour, 06b falls back to the background-task form below and says so in its log.

**Fallback form** (kept in the script either way, chosen by wsm with `coordinator: {harness: claude, arm: task}`): the brief tells the model to run `wsm listen --claim <id> coordinator` with Bash `run_in_background` and to run it again after handling each batch; the prompt check adds a step: held by this session, `state: working` and no live listener pid → print "the coordinator's listener is not running: start it again (Bash, in the background): wsm listen --claim <id> coordinator".

**Process identity.** The lease is keyed on the hook input's `session_id` plus a **verified Claude pid**: walk up from the hook's parent (`ps -o ppid=,comm=,args=`, at most 6 levels) to the first process that is Claude Code: `comm` is `claude`, or its args are a node running `@anthropic-ai/claude-code`'s `cli.js` (an npm install). A shell wrapper (`sh -c` of the shell form, a user's `claude` wrapper script) is skipped. `CLAUDE_PID`, when set, is accepted only if it is that ancestor. No Claude ancestor (the script run by hand, or under some other tool): no claim, exit 0. Its `pstart` is recorded against pid reuse. `session_id` changes on `/clear` and on `/resume` within the same process: SessionEnd releases the old one first, SessionStart claims for the new.

**Subagents.** Only the main session's hooks act: an input with `agent_id` is ignored (subagents share the pid and would otherwise look like a second session).

**Dispatched runs never become coordinator.** A dispatched claude runs with `--setting-sources=` (02d): no project file loads, so none of these hooks run. And twice more: the script exits 0 when `ANFS_RUN` is set, and `wsm lease claim` refuses there (77). Tested in all three places (section 9).

*Alternatives:* the background-task form only (works, but the model must rearm: what the person asked not to rely on); a `Stop` hook that blocks (exit 2 keeps the turn going instead of waiting idle: burns turns); a detached keeper process per lease beside the listener (one more process to supervise; the `working` TTL covers the time between wake and rearm).

## 6. Who writes the settings: wsm, from `space.yml`

**Policy:**

```yaml
coordinator: claude          # or none (the default when absent)
# the long form, when a space wants other wake types or the fallback:
coordinator:
  harness: claude
  wake: [report.filed, run.completed, run.failed, run.canceled, run.rejected, run.auth-required, approval.requested]
  window: 5s
  arm: hooks                 # or task (section 5's fallback)
notify: [approval.requested] # 06c: what anfsd turns into OS notifications ([] for none)
```

**The default wake set** is the list above: a report filed into the space, a run ending (A2A's terminal states) or waiting on the person (04's `run.auth-required`), an approval requested, and unhandled work (02d's `run.rejected`, which 04's inbox shows as an item). Not `run.submitted`/`run.working` (the coordinator's own dispatch noise), not `container.*` (the ui's), not `lease.*`. Unknown keys under `coordinator:` or a value other than `claude`/`none` make wsm warn and treat it as `none`.

**What wsm does at `install`** (and `wsm install` again after `space.yml` changes; `anfs src update` does not run it):

- For **every installed space** it manages `<space dir>/.claude/settings.json`: the inbox line's UserPromptSubmit entry (section 7) always; the five coordinator entries when `coordinator` is `claude`. A space with `coordinator: none` gets only the inbox entry.
- The file is **wsm's**: the hook command is the absolute path of the installed script (`/Users/roberto/.local/lib/anfs/claude/hook`, JSON does not expand `~`), exec form (`args`), no shell. wsm records the sha256 of what it wrote in `$WSM_STATE_HOME/claude/<space id>.sha256`; it rewrites the file only when it is missing or still has that sha, and otherwise refuses for that space with "changed by hand: wsm install -f moves it aside" (the toolkit's `-f` rule). `wsm remove` deletes it when unchanged.
- A space whose **source ships a `.claude/settings.json`** is refused when wsm would manage the file ("one owner: move its keys into space.yml, or set coordinator: none and ANFS_INBOX_HINT=off"): it would otherwise be a link wsm overwrites. None does today (`reviews/claude-settings.md`). `.claude/settings.local.json` is never touched: it is the person's and Claude Code's.
- `wsm ls` shows the file's state per space (`claude: ok`, `changed`, `-`), as it shows links.

**The limit to know:** Claude Code reads project settings from the session's primary working directory only. A session started at `~/spaces/anfs-dev` gets the hooks; one started in `~/spaces/anfs-dev/runtime` does not (that is a git repo of its own, and anfs does not write into clones). `wsm cd anfs-dev` goes to the root, which is how the person starts. Open question 3 covers sessions started deeper.

*Alternatives:* the hooks in the user scope (`~/.claude/settings.json`), owned by core's package, the script resolving the space from `cwd` (works anywhere under a space, but edits the file Claude's `/config` also writes, and runs in every session on the machine; the person chose the space's file); `settings.local.json` (Claude Code writes standing approvals there); a plugin carrying the hooks (enabled from the user scope, so the same ownership question).

## 7. The inbox line, in every session (independent of the lease)

**Recommendation.** The `prompt` entry, after the lease check and in every session of a managed space, coordinator or not, prints one line when the person has inbox items:

```
anfs inbox: 2 waiting here (merge core +3 r-20261024-2, install core/pcm), 1 elsewhere: anfs inbox
```

- **Cheap to read:** anfsd (04a's `internal/land`, extended in 06b) keeps a projection `~/.local/state/anfsd/inbox/summary.json`, rewritten (temp plus rename) whenever an item is requested, answered, expired, dismissed or redispatched: per space id, the count of pending approvals and unhandled items, the oldest's time, and the first two items' WHAT (04's column). The hook reads that one file; with anfsd down it is the last state written (the record is still the inbox's own files).
- **Not on every prompt:** printed when the summary changed since this session last printed it, or every 30 minutes while items wait. The session's last line is kept in `$ANFS_STATE_HOME/claude/<session id>.inbox` (removed at SessionEnd; stale ones pruned by age).
- Off for a session: `ANFS_INBOX_HINT=off`. Nothing is printed when nothing waits.

*Alternatives:* `anfs inbox ls --json` on each prompt (forks the CLI and reads every record: too slow for a 5 s budget on every prompt); the line in the coordinator only (approvals are the person's, and the person may be in any session).

## 8. OS notifications from anfsd (06c)

**Recommendation.** A notification sink in anfsd, a tenant like the report sink: a durable consumer `anfsd-notify` on `anfs.*.anfs.approval.requested` (and any other types a space lists in `notify:`), deliver new, explicit ack.

- **When:** always on `approval.requested` addressed to the person, unless the space says `notify: []`. anfsd cannot know whether the person is away, and does not try: the OS's Focus / do-not-disturb is the person's switch. Host-wide: `ANFSD_NOTIFY=on|off` in anfsd.env (default `on`).
- **Not for `auto`:** the sink waits 10 s after the first event of a space and drops approvals already answered (04's `auto` gives `.requested` then `.given` at once); what remains in that window is one notification per space: "anfs-dev: 2 approvals waiting" / "merge core +3 (r-20261024-2)", body "anfs inbox". At most one per space per minute; the rest are counted into the next.
- **How, without a shell** (fixed argv, the text as an argument, never in a script):
  - macOS: `terminal-notifier -title T -message M -group anfs-<space id>` when it is on PATH; else `osascript -e 'on run argv' -e 'display notification (item 2 of argv) with title (item 1 of argv)' -e 'end run' T M`. anfsd runs as a LaunchAgent in the person's GUI session, where both work.
  - Linux: `notify-send -a anfs T M`. It needs the session bus: systemd `--user` units have `DBUS_SESSION_BUS_ADDRESS` when the user bus exists; when it is missing or `notify-send` is absent, anfsd logs once and `anfs bus status` shows `notify: unavailable (no notify-send)`.
  - `ANFSD_NOTIFY_CMD` (anfsd.env) overrides with an argv whose `{{title}}` and `{{message}}` are replaced, for anything else (ntfy, a phone).
- `anfs bus status` gains a `notify` row (`ok, terminal-notifier, 3 sent today` / `off` / `unavailable`).

*Alternatives:* notifications from the coordinator session (only while a session is open, which is exactly when they are least needed); idle detection (`ioreg` HIDIdleTime on macOS, logind on Linux) to notify only when away (platform code for a guess; Focus does it better); a separate notifier program outside anfsd (another service to install for one consumer).

## 9. Slicing, files, tests

Three plans, each merged alone:

| Plan | What | Repos | After |
| --- | --- | --- | --- |
| **06a** | the mechanism: `lib/anfs/lease.sh` (the file, the lock moved to `files.sh`, live/takeable, claim/renew/release, the cursor and pending), `wsm lease` (list, holder, claim, renew, release), `wsm listen` (cursor, wake set, self-exclusion, window, renewal, holder-death exit, `--hook`, `--max`), `anfs_write_atomic` (temp, fsync, rename), `anfs_events_follow_from` and `anfs_events_cursor_save`, `events ls --cursor` and the `--after` help ("by id order, not log position"), `lease.*` in `ANFS_EVENT_TYPES`, the registry and examples, AGENTS.md ("Leases") | core, events, docs | now |
| **06b** | the Claude adapter and settings ownership: the hook script; wsm writes `.claude/settings.json` (sha ownership, `ls` state, `remove`); `coordinator:` parsed by wsm; anfsd's inbox `summary.json`; the inbox line; anfs-dev's `space.yml` gets `coordinator: claude`; the first task checks the `asyncRewake` timeout cap | core, runtime, anfs-dev, docs | 06a and 04a merged |
| **06c** | anfsd's notify sink, `notify:` in `internal/space`, `ANFSD_NOTIFY`/`_CMD`, the status row; the ui shows a space's coordinator (from `lease.*` events, and `wsm lease ls --json` at start and in the 5-minute reconcile: 03's pattern) | runtime, core, docs | 04a merged (parallel with 06b) |

**Files.** 06a core: `packages/anfs/home/.local/lib/anfs/{lease.sh (new),files.sh,events.sh,report.sh}`, `packages/wsm/home/.local/bin/wsm` (the `lease` group and `listen`), tests (`packages/wsm/tests/lease.bats`, `listen.bats`; `packages/anfs/tests/events.bats` for `--cursor`), AGENTS.md. events: REGISTRY.md, `examples/valid/lease-*.jsonl`, `examples/index.json`, CHANGELOG.md. 06b core: `packages/anfs/home/.local/lib/anfs/claude/hook` (new), `packages/wsm/home/.local/bin/wsm` (settings), tests (`packages/anfs/tests/claude-hook.bats`, `packages/wsm/tests/claude-settings.bats`), AGENTS.md ("`space.yml`": `coordinator:`; "Claude Code settings": what wsm owns); runtime: `go/internal/land` (the summary), README; anfs-dev: `spaces/space.yml`. 06c runtime: `go/internal/notify` (new), `go/internal/space` (`notify:`), `cmd/anfsd`; core: `packages/anfsd/home/.local/bin/anfs-bus` (row), `packages/anfsd/home/.config/anfsd/anfsd.env`, `containers/ui` (the column).

**Test isolation, every bats file and every Go test that touches STATE:** a temp `HOME` **and** every `XDG_*` redirected under it (`XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME`, `XDG_CACHE_HOME`, `XDG_RUNTIME_DIR`), plus `ANFS_*_HOME`, `WSM_*_HOME`, `WSM_SPACES_HOME` and `ANFS_SOURCES_HOME` set explicitly, `ANFS_EVENTS_SPOOL` and `ANFS_RUN` unset, `CLAUDE_*` and `CLAUDECODE` unset unless the test sets them. A guard at the top of the helper fails the run if `ANFS_STATE_HOME` resolves under the real home. (A previous agent damaged live state by redirecting `HOME` alone while `XDG_STATE_HOME` was set.) No test starts a real Claude, touches the running bus, or sends a real notification.

**Fakes.** A fake Claude process: a copy of bash named `claude` in the test's `bin/` (so `comm` is `claude` on macOS and Linux), run as `claude -c '"$HOOK" prompt < "$IN"; rc=$?; exit $rc'` (the trailing statements stop bash from exec-ing the hook, so the hook really is its child). Hook input from fixture JSON (`session-start-startup.json`, `-compact.json`, `prompt.json`, `stop.json`, `session-end-clear.json`, one with `agent_id`). A shell-wrapper chain (fake claude → `sh -c` → hook). Time through one function the tests override (`ANFS_LEASE_NOW`, read only when `ANFS_TEST=1`), so TTLs are tested without sleeping.

**Test spec.**

- **06a lease:** first claim on no file → `lease.claimed from=free`, cursor at the ends; claim when live by another → 75 and the holder on stderr, no event; same holder re-claims → no event, renewed; holder pid dead → taken, `lease.expired why=dead` then `lease.claimed from=dead previous=…`; pid alive with another start time → `why=reused`; renewal older than `ttl` with the pid alive → taken (`why=ttl`); `working` within `work_ttl` → not taken; `renew`/`release` with an old claim → 1 and the file unchanged; `release` keeps the cursor; claim with `ANFS_RUN` set → 77; two claims racing (20 parallel `wsm lease claim` on a free lease) → exactly one claim id in the file, one `lease.claimed`; a stale lock (dead pid inside) is broken; files 0600, dirs 0700; `ls`/`holder` emit nothing.
- **06a listen:** wakes on a listed type, prints the batch, exits 0; with `--hook` the batch on stderr and exit 2; the window gathers events within it, 50 lines then the count; an event by the holder's own actor does not wake; **a line appended late with an older id than the cursor's `last` is delivered** (the regression `--after` would have); a `.bus.jsonl` line likewise; month rollover mid-wait; pending promoted by the same claim, discarded by another (redelivered); renewal written while waiting (fake clock), atomically (a reader polling the file in a loop never sees it partial or missing); **a failing renewal backs off** (the lease directory made read-only: attempts at 30, 60, 120 s then every 5 min on the fake clock, one log line for the failure and one for the recovery, the listener still waiting, and its CPU under the idle bound below while it fails); a cursor saved at an offset mid-file resumes at the next line, not the next id; an offset past the file's end warns once and rereads from 0; the holder pid killed → exit 1 within 2 s, nothing printed; another listener registered → the older exits 1; the claim taken → exit 1; `--max` → 3; idle CPU of a waiting listener under 0.2 s in 10 s (as anfsd's bound), renewals included. `events ls -f --cursor` resumes exactly and advances; **two events appended in the order 10:02 then 10:01 (by id) are both delivered after a cursor taken between them**, where `--after` would print only the first.
- **06a events:** each `lease.*` line valid by `anfs events validate` and under 4096 bytes; the new examples pass `tools/check`.
- **06b hook:** `session-start` in a space claims and prints the brief; a second fake session prints the holder line and does not claim; `compact` re-prints the brief without an event; `prompt` takes a dead holder's lease and prints the takeover brief; `prompt` as holder prints nothing; lost lease prints once; `session-end` releases (`reason=clear` on clear); outside a space, `ANFS_COORDINATOR=off`, `ANFS_RUN` set, an `agent_id` input, or no Claude ancestor → no claim, exit 0; the wrapper chain finds the fake claude; `CLAUDE_PID` naming a non-ancestor is ignored; `listen` for a non-holder exits 0 at once; the parallel SessionStart pair yields one claim; the fallback form's prompt line when the listener is gone.
- **06b wsm settings:** `install` writes the file with the absolute script path and the five entries for `coordinator: claude`, only the inbox entry for `none` or absent; rewriting is idempotent (same sha); a hand-edited file is refused without `-f` and moved aside with it; a source-shipped `.claude/settings.json` is refused; `remove` deletes an unchanged file and leaves a changed one; `settings.local.json` never touched; `ls` shows the state; the JSON is valid (`jq -e` in the test only) and its `asyncRewake`/`timeout` keys as specified.
- **06b inbox:** the line from a fixture `summary.json`; printed on change and after 30 min, not otherwise; `ANFS_INBOX_HINT=off`; none when nothing waits. runtime: the summary rewritten on request, answer, expiry, dismiss and redispatch (04a's test fakes), atomically.
- **06b dispatched runs:** runtime golden: the claude argv still carries `--setting-sources=` (02d's golden, asserted again with a space whose `coordinator: claude`); core: the two `ANFS_RUN` refusals above.
- **06c:** the sink with a fake notifier (an argv recorder): one notification per space in the window; an `auto` approval (`.requested` then `.given`) gives none; `notify: []` gives none; `ANFSD_NOTIFY=off` gives none and no consumer; rate of one a minute; terminal-notifier preferred when on PATH, osascript otherwise, notify-send on Linux; the text passed as arguments, never through a shell (a title with quotes and `$(...)` arrives intact); unavailable notifier logged once. ui rspec: the coordinator column from `lease.*` events.

## By hand, for the person (no migration code)

After **06a**: `anfs src update core`; `ppm install core/anfs core/wsm`. In anfs-dev: `wsm lease ls` (all free); optionally `wsm lease claim coordinator --kind shell` then `wsm listen --claim <id> coordinator` in a terminal, file a test report from another, see it printed; `wsm lease release --claim <id> coordinator`.

After **06b** (04a merged and installed first):

1. `anfs src update core anfs-dev`; rebuild and restart anfsd (the summary); `ppm install core/anfs core/wsm core/anfsd`; `wsm install anfs-dev` (writes `~/spaces/anfs-dev/.claude/settings.json`; check `wsm ls anfs-dev` says `claude: ok`).
2. Stop today's hand-run listener: end the background `anfs events ls -f ... | grep -m1 ...` task in the anfs-dev coordinator session, and drop the instruction to rearm it from wherever it lives (the session's notes or memory).
3. Start Claude Code at `~/spaces/anfs-dev`; accept the workspace trust dialog if it asks (hooks wait for it). It should print the brief. `wsm lease holder coordinator` names it, `listening`.
4. Start a second session there: it names the first as coordinator. File a test report from cnfs (`anfs report` there): the first session wakes within seconds with the batch.
5. Quit the first; type in the second: it says it took over (`from=dead` or `released`), and an event filed meanwhile is in its first batch.
6. Other spaces: they get the inbox line at their next `wsm install`; turn a coordinator on by adding `coordinator: claude` to their `space.yml` (in their source, your commit) and `wsm install <space>` again.

After **06c**: rebuild and restart anfsd; `ppm install core/anfsd`; `brew install terminal-notifier` if you want its look (optional); `anfs bus status` shows `notify ok`; the next approval request pops a notification. On a Linux host: check `notify-send` exists and `systemctl --user show-environment | grep DBUS`.

## Out of scope

- The dispatched `<space>/coordinator` role (unattended triage): a 02b dispatch rule with `as: coordinator` and its prompt, when the person wants one; 06 only keeps runs from holding the interactive lease.
- Adapters for other harnesses (opencode, pi) and for herdr or Firstmate as holders: the lease and listener are generic; their adapters are their packages' business.
- Sessions started below a space's root getting the hooks (open question 3), and a per-space coordinator brief file (the space's CLAUDE.md already reaches every session).
- `ANFS_AS=coordinator` for the coordinator session's own events (its `actorrole`): with roles.
- Idle detection for notifications; notifications for anything but `notify:`'s types; a phone channel beyond `ANFSD_NOTIFY_CMD`.
- Leases across hosts (a NATS KV or a durable consumer), and authentication of holders (one host, one uid, as 04).

## Open questions for the person

1. **How the listener is armed:** `asyncRewake` hooks on Stop and SessionStart, so a wake never depends on the model (section 5), or the background task the model starts (today's way, with the prompt hook reminding it). **Pick: hooks**, if 06b's first task finds an `asyncRewake` timeout of at least an hour accepted; otherwise the task form, which the script also carries (`arm: task`).
2. **The cursor:** byte offsets per month file beside the lease (the shipper's shape), the last id for display only, at-least-once with the pending step (section 3), against id order (`--after`: loses late lines from runs, the recorder, other writers and clock skew) or a NATS durable consumer (deaf when the bus is down). **Pick: byte offsets in files** (agreed 2026-10-24 with the person and Claude Desktop's review).
10. **Renewal failures:** back off (30 s, 60 s, 120 s, then every 5 min) and let the lease expire if they outlast `ttl`, against stopping the listener at the first failure (one bad write would end the coordinator). **Pick: back off** (agreed 2026-10-24).
3. **Sessions started below the space's root** (`~/spaces/anfs-dev/runtime`) read no space settings (Claude Code reads project settings from the starting directory only). Accept it (start at the root, as `wsm cd` does), or later add a user-scope hook owned by core's package that resolves the space from `cwd`. **Pick: accept for 06**; revisit if it bites.
4. **TTLs:** `ttl` 90 s with a renewal every 30 s while listening, `work_ttl` 2 h while a woken or prompted session works. **Pick: these defaults**, overridable per claim; a turn longer than 2 h loses the role to the next session that prompts, which is acceptable for a companion.
5. **The inbox line in spaces without a coordinator:** wsm then manages `.claude/settings.json` in every installed space. **Pick: yes**, every installed space gets the inbox entry; `ANFS_INBOX_HINT=off` per session.
6. **The default wake set:** `report.filed`, the terminal `run.*` states, `run.auth-required`, `run.rejected`, `approval.requested`. **Pick: as listed**; a space narrows or widens it with `coordinator: {wake: [...]}`.
7. **Notifications:** always on `approval.requested` (not answered within 10 s), per-space `notify:` list, host switch `ANFSD_NOTIFY`; no idle detection. **Pick: as described**; unhandled work (`run.rejected`) not notified by default.
8. **Runs and leases:** `wsm lease claim` refuses in a dispatched run (77). **Pick: refuse**; a run that should hold a role is a different design (a long-lived worker), not a task.
9. **Where the lease commands live:** in wsm (`wsm lease`, `anfs space lease` through the alias) with the library in core/anfs, against a new top-level `anfs lease`. **Pick: wsm**, beside `wsm events`.

## Sources checked (2026-10-24, read-only)

- https://code.claude.com/docs/en/hooks.md: the common input fields; SessionStart's `source` values, stdout and `additionalContext` injection, `CLAUDE_ENV_FILE`; UserPromptSubmit's 30 s default timeout and that its stdout and `additionalContext` are each injected as a system reminder; SessionEnd's reasons, no blocking, the 1.5 s budget raised by a per-hook `timeout` up to 60 s; "Run hooks in the background": `async`, timeout not enforced on async hooks but enforced on `asyncRewake`, `-p` kills them at teardown, and "an `asyncRewake` hook that exits with code 2 wakes Claude immediately even when the session is idle"; hooks run in parallel, "in their own session without a controlling terminal", `sh -c` or exec form; workspace trust before any settings-file hook in an interactive session. The page was read through a fetch tool in three parts; the quotes are as it returned them.
- https://code.claude.com/docs/en/settings.md: project `.claude/settings.json` is read from the session's primary working directory; Claude Code writes standing approvals to `.claude/settings.local.json`; hooks edits reload in a running session.
- **Not confirmed:** the largest `timeout` an `asyncRewake` hook accepts; whether async hooks of an interactive session are killed at its exit; whether a SessionStart hook may be `asyncRewake` (the page lists `async` for any command hook). 06b's first task checks all three.
- Installed Claude Code 2.1.296: `asyncRewake`, `CLAUDE_CODE_SESSION_ID` and `CLAUDE_ENV_FILE` are in the binary's strings; this session's own process tree is `claude` (comm `claude`) → `/bin/zsh` → the command, and its environment has `CLAUDE_PID` and `CLAUDE_CODE_SESSION_ID`.
- Code read, not run: core `a90652c` (`events.sh`: `--after` compares ids; follow keeps descriptors per file; `_anfs_ev_actor`), runtime `367ee0c` (`supervise.alive`, `psStart`; `space`: `dispatcher:`), wsm (`_mf_plan`, `_home_ok`, `.claude/skills` written by the skills CLI in anfs-dev's space directory), events' REGISTRY.md.
