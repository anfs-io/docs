# Plan 02d — safe to turn on dispatch

Written 2026-10-24 against runtime main `ae71386` (02c), core and anfs-dev main, after the review
(`AGENT-NATIVE-REVIEW.md`, 5.2) and `reviews/claude-settings.md`. Part of [plan 02](plan.md).
Mergeable alone. After it, a dispatched run's own events reach the bus under the run's identity,
a run gets no credential anfsd happens to hold, a dispatched Claude Code loads only what anfs hands
it, a full queue or a spent rate delays a report instead of dropping it, and a space can turn
anfsd's dispatcher off. That is the bar for turning on anfs-dev's two rules, which stay `enabled:
false` until the person does it by hand. Open questions answered 2026-10-24: all as recommended.

The review's findings this builds, checked against the code (2026-10-24):

- **F1 is real.** claude's `sandbox.filesystem.allowWrite` is the job, its TMPDIR and `access:`'s
  write paths (`supervise.execute`, the `write` list; `harness/claude.go`). `anfs_emit` appends to
  `$XDG_STATE_HOME/anfs/events/<scope>/<month>.jsonl`, which is outside that list, so a sandboxed
  `anfs report ...` loses its event and only prints a warning.
- **F5 is real.** `Reject` writes a record, and `Has` (`byEvent`, any state) answers "already
  handled" for it from then on. A `report.filed` that arrives while the maintainer's queue is full
  is never dispatched.
- **F7 is real.** `baseEnv` passes anfsd's whole environment through, minus seven anfs names.
- **F6** (self-asserted actors) is low on its own. It costs nothing extra once F1 routes a run's
  events through anfsd, so 02d does it in the same place.

## Context: read these first

The job is `anfs work new agents-02d runtime core docs`.

- docs: `AGENT-NATIVE.md` ("Decided"), `AGENT-NATIVE-REVIEW.md` (4.1, 4.3, 5.2), `reviews/claude-settings.md`,
  [plan.md](plan.md), [02b](02b-dispatch.md) and its log (the decisions there, especially 1, 6 and 9)
- runtime: `go/internal/supervise` (`baseEnv`, `execute`, `prepare`, `end`, `finishDead`,
  `Start`'s adoption, `Has`/`index`/`byEvent`, `Full`), `internal/dispatch` (`Decide`, `Handle`),
  `internal/harness`, `internal/cloudevent`, `internal/spool`, `internal/shipper`, `internal/space`
- core: `packages/anfs/home/.local/lib/anfs/events.sh` (`_anfs_emit`, `ANFS_EVENTS_HOME`),
  `report.sh`; `packages/anfsd/home/.local/bin/anfs-agents`; AGENTS.md "Dispatch", "Events"

## 1. A run's events: a per-run spool (F1, F6)

**Decision: each run writes its events into its own directory, and anfsd moves them into the log
under the run's identity.** We do not open the shared events directory to the sandbox. A writable
shared log would let a run write lines for any space under any actor, and would let it truncate
or rewrite other files there, since "append-only" is only a convention.

- **The directory.** `~/.local/state/anfsd/runs/<run id>/events/` (0700), made by anfsd before
  the harness starts. It uses the log's layout: `events/<scope>/<YYYY-MM>.jsonl`. Only this
  subdirectory is writable by the run. `run.json`, `event.json`, `prompt.md`, `claude.json`,
  `exit`, `cancel`, `out.log` and the cursor file stay unwritable:
  - claude: the directory is added to `sandbox.filesystem.allowWrite` only, **not** to the
    `Edit(...)`/`Write(...)` allows. Only Bash (and so `anfs_emit`) can write there; Claude's
    file tools cannot.
  - opencode: an `external_directory` allow (one permission for read and write, as for `access:`).
  - antigravity: `--add-dir`.
  - pi: nothing to hand over (unenforced, as before).
  - `harness.Spec` gains `Spool string`, kept apart from `Write`, so claude can treat it
    differently. The footer gains one line: "Events you emit go through `$ANFS_EVENTS_SPOOL` and
    are recorded as yours (run `<id>`)."
- **The variable: `ANFS_EVENTS_SPOOL=<run dir>/events`.** The brief's example was
  `ANFS_EVENTS_DIR`; `ANFS_EVENTS_SPOOL` is used because the variable changes where emits are
  written and nothing else. `events ls` and `report wait` inside the run keep reading the real
  log, which a name ending in `_DIR` would suggest it changes.
  `lib/anfs/events.sh`: when it is set, `_anfs_emit` writes to
  `$ANFS_EVENTS_SPOOL/${sid:-host}/<month>.jsonl` instead of `$ANFS_EVENTS_HOME/...`. Everything
  else (scope, actor, the one `O_APPEND` write) stays the same, except that the size limit drops to
  4096 − 512 bytes, which leaves room for the stamp. `ANFS_EVENTS=off` still wins. A run reads
  its own events in the log only after anfsd has ingested them, a second or so later.
- **Ingestion** (`internal/supervise/ingest.go`):
  - Each run's spool is a `spool.Spool` with `Dir=<run dir>/events` and its cursors in `<run
    dir>/cursors.json`, under the same discipline as the month files: complete lines only, a
    cursor per file, temp file + fsync + rename, and only after the lines are written.
  - A `spool.Watch` runs on the directory while the run works, with fsnotify and the 30 s
    reconcile, and stops after the final drain. There is no timer when no run is working.
  - For each line:
    1. `cloudevent.Parse`. An invalid line is logged once (run, file, offset) and stepped past.
    2. **Refused** (logged, counted in `run.json` as `events_refused`): `run.*`, which is anfsd's
       own; a scope directory that is not the line's `source`; and anything past
       `ANFSD_RUN_EVENTS_MAX` (anfsd.env, default 1000 lines per run). Symlinks are never read:
       `spool.Files` takes only regular files and real directories.
    3. **Stamped**, by a new `cloudevent.Stamp(line, Stamp{...})`, which rewrites only these
       attributes and keeps the rest byte-identical in the profile's order, `data` last (the bash
       reader relies on that):
       - `actor` = `<harness>:<last 8 of the run id>` (what `ANFS_ACTOR` already says);
       - `actorspace` = the rule's space id;
       - `actorrole` = the role;
       - **`actorrun`** = the full run id (new extension);
       - **`actorclaimed`** (new extension), set only when the line's own `actor`, `actorspace`
         or `actorrole` differs from the stamp. Its value is `<actor>`, plus ` as <actorspace
         id>/<actorrole>` when the line named a role, e.g. `shell:roberto as tkj76y3nfx/person`.
       - `id`, `time`, `source`, `type`, `subject` and `data` are kept.
    4. **Appended** to `$XDG_STATE_HOME/anfs/events/<scope>/<the month of the line's time>.jsonl`.
       This is the log the shipper already follows, so "the files are the complete record" still
       holds and nothing else changes. It is one `O_APPEND` write, through a new
       `cloudevent.AppendLine`. A stamped line over 4096 bytes is refused and counted (the
       512-byte reserve makes that a forged line, not a real one).
- **Forged claims: kept visible, not overwritten silently, not rejected.**
  - Rejecting would drop real work from honest tools whose actor logic differs. For example, a git
    hook in the job that emits with `CLAUDECODE=1` gets `claude:<session>`, not `ANFS_ACTOR`.
  - Overwriting silently would hide an attempt to pass as the person.
  - The cost is one extension that readers learn: `actorclaimed` is a claim and never an identity.
    The actor is the stamp. 05's spec registers `actorrun` and `actorclaimed` (a note for 05;
    02d does not edit it).
  - On this host, anfsd now attests the actor of everything a run writes through its spool. That
    is F6's "the publisher attests the actor", which cnfs and lgat are told to build on.
  - The `loop` guard uses `actorrun` when it is present, so a run can no longer slip past it by
    claiming another actor.
- **When it drains:**
  - **while the run works**, on each watcher wake-up;
  - **at the end**, in `end()` before the final `run.*` is emitted, for runs that started
    (`completed`, `failed`, `canceled` after start). A run's own events therefore come before its
    `run.completed` in the month file;
  - **in `finishDead`**, before the run is ended from its exit file, or as `orphaned`;
  - **at adoption**: `Start` drains each adopted run at once and then watches it. It also drains
    every run that ended within the last 24 h whose spool files are past their cursors (lines
    written between the end and a crash, or by a straggling child before `killLater`). Lines
    written later than that stay in the run's directory; `anfs agents show` reports "N bytes not
    ingested".
- **No duplicates, in the files or on the bus.**
  - A crash between the append and the cursor's rename can only repeat the last batch (at most
    256 lines). So the first batch per run after anfsd starts reads the last 1 MiB of each month
    file it appends to and skips ids already there. That bound covers a full batch.
  - On the bus, the shipper publishes with `Nats-Msg-Id` = the event id, which the run's emitter
    chose and anfsd kept. The stream's 24 h duplicate window drops a resend.
  - A run that replays an existing event's id gets its line dropped on the bus as a duplicate.
    That is harmless: it can suppress nothing that was not already shipped.
- **Not closed here.** `anfs report new` from inside a run still cannot write its record
  (`$ANFS_STATE_HOME/reports/`), which is outside the sandbox. The maintainer's prompt does not
  file reports. Out of scope; see the end of this plan.

## 2. The environment: an allow-list (F7)

`baseEnv` becomes `runEnv(harness)`. Nothing passes through unless it is named.

| From anfsd's environment, when set | Why |
| --- | --- |
| `PATH`, `HOME`, `USER`, `LOGNAME`, `SHELL`, `LANG`, `LANGUAGE`, `LC_*`, `TZ`, `TERM`, `__CF_USER_TEXT_ENCODING` | a working shell |
| `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME`, `XDG_CACHE_HOME`, `XDG_RUNTIME_DIR`, `XDG_CONFIG_DIRS`, `XDG_DATA_DIRS` | the toolkit's paths |
| `ANFS_*`, except `ANFS_WORK`, `ANFS_WORK_SAID`, `ANFS_EVENTS_SPOOL` and the per-run names below | anfs's own settings (`ANFSD_*` are the daemon's and do not pass) |
| the harness's credential: claude `ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`; opencode, pi, antigravity none (they read their own auth files) | the harness must reach its model |

| Set by anfsd | |
| --- | --- |
| `TMPDIR`, the job's variables (`anfs work env`) | as 02b |
| `ANFS_ACTOR`, `ANFS_AS`, `ANFS_RUN`, `ANFS_EVENT`, `ANFS_EVENTS_SPOOL` | the run's identity and spool |
| `GIT_TERMINAL_PROMPT=0`, `GIT_CONFIG_COUNT`/`GIT_CONFIG_KEY_n=credential.helper`/`GIT_CONFIG_VALUE_n=` (empty) | git asks nobody, and the empty value resets every credential helper from the system and global config, including macOS's `osxkeychain`. `n` follows any `GIT_CONFIG_*` the job already sets |
| claude: `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` | Claude Code strips its own credential from every subprocess (hooks included) |
| the harness's own (`OPENCODE_CONFIG_CONTENT`, ...) | as 02b |

So `SSH_AUTH_SOCK`, `GH_TOKEN`, `GITHUB_TOKEN`, `GIT_ASKPASS`, `SSH_ASKPASS`, `AWS_*`, `NPM_TOKEN`
and anything else anfsd's environment holds never reach a run. anfsd's own `anfs work new|path|env`
calls use the same list (`anfs work new` makes worktrees from local sources; the job's first task
confirms it needs no network, and if it does, those calls keep anfsd's full environment and
only the harness gets the list).

**What is enforced, and what is only documented.** An environment variable is a convenience the
agent can undo (`export`, `env -u`). The real stops are elsewhere, and only claude has them:

| Path to a credential | claude (sandboxed Bash) | opencode, pi, antigravity |
| --- | --- | --- |
| `SSH_AUTH_SOCK`, tokens in the environment | **enforced**: not in the environment; the harness key is also unset in Bash by `sandbox.credentials.envVars` `deny` | removed from the environment (undoable only if the value is found elsewhere) |
| key and token files: `~/.ssh`, `~/.config/gh`, `~/.git-credentials`, `~/.netrc`, `~/.claude/.credentials.json`, `~/.local/state/nats` (the bus's passwords: a run with them could publish past the stamp) | **enforced**: `sandbox.filesystem.denyRead` (the Read tool is already limited to the read paths by `dontAsk`) | readable: **documented** |
| macOS keychain (git's `osxkeychain`, `gh`'s token, `security`) | helper reset by the env (a hint), and **no network out of the sandbox**: under `dontAsk`, with `allowedDomains` empty, every host is refused (Claude Code's sandboxing docs), so a token taken from the keychain cannot be used for push | reachable: **documented** |
| `git push`, `gh` over the network | **enforced** by the sandbox's network (and denied by rule, as before) | denied by rule only (opencode), or not at all (pi, antigravity): **documented** |

This is why anfs-dev's rules use claude and why `anfs agents rules` marks the others `partial` or
`unenforced` (02b). AGENTS.md's "Dispatch" says it in one table. A side effect worth knowing:
claude's sandboxed Bash has no network, so a fix that needs a module download fails. That is
02b's state too, and it is now said in the footer and AGENTS.md. A `network:` list per space is
later work.

## 3. What a dispatched harness loads (settings isolation)

Checked against Claude Code's docs (cli-reference, headless, settings, authentication, sandboxing;
2026-10-24):

- `--setting-sources <user,project,local>` picks which file scopes load. `--settings` is the
  command line and always loads.
- `--strict-mcp-config` connects only `--mcp-config` servers. With none given, no servers connect,
  including the user's in `~/.claude.json`, which is not a setting source.
- **`--bare` skips hooks, skills, plugins, MCP, auto memory and CLAUDE.md, but "never reads OAuth
  credentials or the system keychain", and does not read `CLAUDE_CODE_OAUTH_TOKEN`.** Only
  `ANTHROPIC_API_KEY` or an `apiKeyHelper` work. The person's runs use the subscription login
  (02b's by-hand step 3), so `--bare` alone would fail every run at its first request.

**Decision:**

- claude always gets `--setting-sources local --strict-mcp-config`. A job's cwd has no
  `.claude/settings.local.json`, so in effect that means none (an empty list is not documented;
  the job's first task checks, from `--help` and the binary, whether `--setting-sources ''` is
  accepted, and uses it if so).
- The run's `claude.json` gains `claudeMdExcludes: ["<HOME>/.claude/CLAUDE.md"]`, plus the
  sandbox keys from 2 (`denyRead`, `credentials.envVars`) and the spool's `allowWrite`.
- The result: no user or project hooks (herdr's, a repo's untrusted ones), no user `env`, no
  plugins (they are enabled from the user scope), no MCP servers, and not the person's own
  CLAUDE.md. The repos' CLAUDE.md and AGENTS.md still load.
- **`ANFSD_CLAUDE_BARE=on`** (anfsd.env, default `off`) adds `--bare`, for a host whose claude
  authenticates with `ANTHROPIC_API_KEY` or an `apiKeyHelper`. That is the strict mode: no user
  skills or agents, and no auto memory. It is one flag and one golden file.
- Hooks a role needs come from the run's `--settings` and only from there. 02d adds no `hooks:`
  key; that is the place for one when a role needs it.
- What still loads without `--bare`: user skills (`~/.claude/skills`) and user subagents
  (`~/.claude/agents`). Both are read-only definitions the agent may choose to use, not code that
  runs by itself, so this is accepted and written in AGENTS.md.

The other harnesses:

| Harness | Added | Status |
| --- | --- | --- |
| pi | `--no-extensions --no-mcp --no-skills --no-prompt-templates --no-approve` (`pi --help`, 1.1.0, checked here 2026-10-24). Context files (AGENTS.md) stay | isolated (still `unenforced` for access) |
| opencode | `OPENCODE_DISABLE_PROJECT_CONFIG=1`, `OPENCODE_DISABLE_CLAUDE_CODE=1` (stops it reading `~/.claude`'s CLAUDE.md and skills), `OPENCODE_DISABLE_EXTERNAL_SKILLS=1`: names from its binary's strings (1.18.35). The global `~/.config/opencode` (plugins, MCP) has no switch found; `OPENCODE_CONFIG_DIR`'s meaning (replace or add) is the job's first task to check | partial, documented |
| antigravity | none known (its `--help` hangs) | documented: not isolated |

Golden files: `claude.golden` (argv: the two flags; `claude.json`: `claudeMdExcludes`, `denyRead`,
`credentials.envVars`, the spool in `allowWrite` and not in `Edit`/`Write`), `claude-bare.golden`
(new), `pi.golden`, `opencode.golden` (its env), `antigravity.golden` (the spool's `--add-dir`).
`anfsd match` and `anfs agents match` print the new argv and environment.

## 4. Backpressure: delay, with bounds (F5)

**Decision:** `queue` and `rate` **hold** a run by default instead of rejecting it. A rule's
`overflow:` can ask for `reject`, and a rule's `hold:` caps how many runs may be held. `stale` and
`loop` stay terminal. A rejection for `queue` or `rate` does not count as handling the event.

- **Format** (per rule, beside 02b's keys):

  ```yaml
  dispatch:
    - name: fix-core-report
      ...
      overflow: delay      # delay (default) | reject: what a full queue or a spent rate does
      hold: 20             # most runs of this rule held at once (default 20, 0 to 200)
  ```

- **Held** is not a new state. A held run is `submitted` (A2A's "received, not started"), and its
  record has `held: rate|queue` and `held_since`. `run.submitted` is emitted once, and **its data
  carries the hold**: `held` (`rate` or `queue`) and `detail`, the reason (e.g. "20 runs in the
  last 1h (rate: 20/h)"); a run not held has neither key. Leaving hold emits nothing; the next
  event is `run.working`, or a terminal state.
  - Why not `run.delayed`: the `run.*` types are A2A's task states (plan.md 3), and 04, 05 and the
    ui are built on that set. "Waiting" is `submitted` with a reason. A2A has no delayed state,
    and a type outside the set is one more thing every reader must learn.
- **The guards in `Decide`**, in order:
  1. invalid job
  2. stale: terminal
  3. loop: terminal (moved before rate and queue, so a loop is never held)
  4. rate
  5. queue
  - When rate or queue trips: under `overflow: reject`, `run.rejected` as before. Under `delay`,
    the run is held if the rule holds fewer than `hold:` runs. Otherwise `run.rejected`, reason
    `rate` or `queue`, with detail `hold: 20 full`: visible, and refused.
  - A rule that already holds runs holds every new one too (first in, first out), so a later
    event never overtakes an earlier one.
- **Leaving hold** (the scheduler):
  - A `queue`-held run moves into its role's queue when the queue has room (on each `schedule()`).
  - A `rate`-held run moves in when its rule's window frees a start. There is one timer per rule
    while it holds runs, set to the oldest counted start plus the window, and no polling.
  - `rate` now counts runs as they **enter the queue** (record `queued`), not as they are
    submitted, so a held run does not use up the rate while it waits.
- **Stale is checked again when a run starts.** Before `execute`, any run that waited past its
  rule's `stale` (the record gains `stale`) ends `run.rejected reason=stale`, terminal. This covers
  held runs and a long ordinary queue alike. A redispatched run is exempt.
- **Handled.** `Has` becomes `Seen` (any record, so a NATS redelivery is acked without a second
  `run.rejected`, as 02b log 9) and `Handled`. An event is **handled** by a rule when it has a
  run that is not `rejected` for `queue` or `rate`. Unhandled events are what `redispatch` takes
  and what `anfs agents ls --unhandled` lists (`anfs inbox` shows them in 04).
- **Survives a restart.** Held runs are records (`submitted` + `held`). `Open` puts them back in
  hold in id order, and the rate timers are recomputed from the records.
- **`anfs agents redispatch <event-id> [rule] [-y]`** (core/anfsd's `anfs-agents`):
  - It finds the event's line in the log: the id's UUIDv7 milliseconds give the month, then
    `grep -F` in each scope's file.
  - It shows `anfs agents match --event -` (what would happen), asks without `-y`, writes
    `~/.local/state/anfsd/redispatch/<uuid>.json` (`{event: <line>, rule: "" | name, by:
    <actor>}`) and sends SIGHUP to anfsd's pid. This follows `stop`'s pattern: a file, then a
    signal.
  - anfsd reads the directory at start and on SIGHUP (which also reloads the rules, as now).
  - For each rule (or only the named one) that matches and has no handled run for the event, it
    submits a run that **skips stale, rate and loop**: the person asked for it. The run can still
    be held for `queue`. The record gets `redispatch_by`, and `run.submitted` data
    `redispatch=<by>`.
  - A rule that is disabled, or whose space has `dispatcher: none`, is refused. So is an event
    that already has a working or completed run, unless `--again` is given (for a failed run's
    second try by hand).
  - The request file is removed when it has been handled, and the result is logged.
  - With anfsd down, the request waits for its start, and the CLI says so.
- **Visible:**
  - `anfs agents ls` STATUS `held:rate` or `held:queue`.
  - `anfs agents ls --unhandled`: `EVENT SUBJECT RULE REASON AGE`.
  - `anfs bus status`'s dispatch row adds "N held".
  - `anfs agents rules --json` gains `overflow` and `hold`.

## 5. Turning anfsd's dispatcher off for a space

**Decision: a top-level `dispatcher: anfsd | none` in `space.yml`** (absent: `anfsd`).

- `dispatch: off` was the alternative. `dispatch:` is the list of rules, and a key that can be a
  list or a word makes the one parser two-typed.
- `dispatcher:` reads as the review's `tools: dispatcher:` slot (4.3) will. When slots arrive, it
  moves there by a by-hand edit (no migration code).
- Only `anfsd` and `none` are accepted. A named tool (`firstmate`) waits for the slots; until then
  another tool simply means `none`.

With `none`:

- The space's rules are still parsed and validated, and shown with STATUS `off` ("dispatcher:
  none") in `anfs agents rules`, `anfsd rules` and `match`.
- They are not loaded and are left out of `anfsd-dispatch`'s filter subjects.
- Runs of that space still queued or held at the reload end `run.canceled by=dispatcher:none`.
  Working runs go on (`anfs agents stop` stops them).
- `redispatch` refuses the space.
- The shipper and the report sink are unaffected. They are not dispatch, and a space that hands
  dispatch to another tool still wants its events shipped and its BACKLOG rendered.

This is the review's acceptance test (4.1), "turn off anfsd's dispatcher for one space, let
another tool handle it, change no core code", minus the other tool.

## Files

- **runtime:**
  - `go/internal/supervise`: `runEnv`; `ingest.go`; `execute`, `end`, `finishDead` and `Start`
    drain; held runs, `Seen`/`Handled`, stale at start, redispatch requests; `record.go` fields
    `held`, `held_since`, `queued`, `stale`, `events_ingested`, `events_refused`,
    `redispatch_by`.
  - `internal/dispatch`: guard order, overflow, hold, `dispatcher: none` in the filters.
  - `internal/space`: `overflow:`, `hold:`, `dispatcher:`; status `off`.
  - `internal/harness`: `Spec.Spool`, the claude flags and settings keys, `ANFSD_CLAUDE_BARE`,
    pi's flags, opencode's env, the golden files, and the footer line.
  - `internal/cloudevent`: `Stamp`, `AppendLine`, and `ActorRun`/`ActorClaimed` read by `Parse`.
  - `internal/config`: `ANFSD_RUN_EVENTS_MAX`, `ANFSD_CLAUDE_BARE`.
  - `cmd/anfsd`: `match` and `rules` output.
  - `testfake`: a harness that runs a given script.
  - `test/e2e.bats`, README, AGENTS.md.
- **core:**
  - `lib/anfs/events.sh`: `ANFS_EVENTS_SPOOL` and the reserve.
  - `packages/anfsd/home/.local/bin/anfs-agents`: `redispatch`, `ls --unhandled`, the `held` and
    `off` statuses.
  - `anfs-bus`: the dispatch row's held count.
  - `anfsd.env`: `ANFSD_RUN_EVENTS_MAX`, `ANFSD_CLAUDE_BARE`.
  - AGENTS.md: "Dispatch" (the run's spool and the stamp, the env allow-list, the enforcement
    table, settings isolation, `overflow:`/`hold:`, `redispatch`, `dispatcher:`); "Events" (the
    `actorrun` and `actorclaimed` extensions); "`space.yml`".
  - Tests.
- **docs:** `02d-log.md`.
- **anfs-dev:** nothing. The rules are turned on by hand, below.

## Order of the work (commits, each green on its own)

0. **First task (no harness started; `--help`, binary strings and the docs only).**
   - claude, the installed version: `--setting-sources` (does `''` work?), `--strict-mcp-config`,
     `--bare`; the keys `claudeMdExcludes`, `sandbox.filesystem.denyRead`,
     `sandbox.credentials.envVars` in its binary's strings; `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB`.
   - opencode: what `OPENCODE_CONFIG_DIR` and `OPENCODE_DISABLE_*` do (its docs or strings).
   - pi: the flags above.
   - That `anfs work new` makes a job with no network.
   - What does not hold is a finding in the log, and section 3 is corrected.
1. **core: `ANFS_EVENTS_SPOOL`** in events.sh, with bats. It comes first because the runtime e2e
   in 3 can use it.
2. **runtime: the env allow-list** (`runEnv`, git's helper reset), with tests. It is small,
   touches only `prepare` and `anfs()`, and gives 3 its place for `ANFS_EVENTS_SPOOL`.
3. **runtime: the per-run spool, ingestion and the stamp**: `cloudevent.Stamp`/`AppendLine`,
   `ingest.go`, the drains, `Spec.Spool` in every harness, golden files, and the e2e.
4. **runtime: settings isolation**: the claude flags and `claude.json` keys, `ANFSD_CLAUDE_BARE`,
   pi and opencode, golden files.
5. **runtime: backpressure**: format, guard order, held runs, `Seen`/`Handled`, stale at start,
   redispatch requests, the restart.
6. **runtime: `dispatcher: none`.**
7. **core: `anfs agents redispatch`, `ls --unhandled`, statuses, `anfs bus status`**, AGENTS.md,
   tests.
8. **docs: `02d-log.md`.**

## Test Spec

runtime (Go; the fakes as in 02b, never a real harness):

- **env:**
  - Setup: a fake harness writes `env` to a file. The supervisor's `Env` holds `SSH_AUTH_SOCK`,
    `GH_TOKEN`, `GITHUB_TOKEN`, `AWS_SECRET_ACCESS_KEY`, `ANTHROPIC_API_KEY`, `ANFSD_MAX_RUNS`
    and `ANFS_SOURCES_HOME`.
  - The run's environment is exactly the allow-list plus anfsd's names: `ANTHROPIC_API_KEY` only
    for claude, `ANFS_SOURCES_HOME` passes, `ANFSD_*` does not.
  - `git config --get-all credential.helper` inside the run prints nothing, with a `HOME`
    gitconfig that sets `osxkeychain`, and with a job env that already sets `GIT_CONFIG_COUNT=1`.
  - `anfs()`'s calls get the same list.
- **ingestion:**
  - A fake harness appends three lines to `$ANFS_EVENTS_SPOOL/<sid>/<month>.jsonl`:
    1. honest (`ANFS_ACTOR`, `ANFS_AS`);
    2. forged (`actor: shell:person`, `actorrole: person`);
    3. `run.completed`.
  - Expected in the space's month file: (1) stamped, with `actorrun` and no `actorclaimed`; (2)
    stamped, with `actorclaimed: "shell:person as <sid>/person"`; (3) not there and counted.
  - Ids are kept and `data` is last; a byte-for-byte golden of a stamped line.
  - The run's events come before its `run.completed` in the file.
  - A partial line waits for its newline.
  - A scope directory that does not match the line's `source`, a symlinked file, and line 1001
    are refused.
  - A line that would exceed 4096 bytes once stamped is refused.
- **drains:**
  - At a normal end.
  - In `finishDead`: the run left an exit file and a line while anfsd was down.
  - At adoption: anfsd restarted mid-run, the run then writes one more line and ends; both
    lines are in the log once.
  - **Crash between append and cursor**: the cursor file is put back, then anfsd starts. The
    month file has each id once (the tail check), and the shipper on embedded JetStream publishes
    each once.
- **harness:**
  - Golden files for all five.
  - claude: the spool in `allowWrite`, not in `Edit`/`Write`; `denyRead` holds the six paths;
    `credentials.envVars` denies the harness keys; `--setting-sources local --strict-mcp-config`;
    with `ANFSD_CLAUDE_BARE=on`, `--bare` too.
- **backpressure:**
  - Queue full with `delay`: held, `run.submitted` with `held=queue`, started when a slot frees,
    in order.
  - `overflow: reject`: `run.rejected`, and the event is unhandled.
  - `hold: 2` with a third: `run.rejected` with detail `hold: 2 full`.
  - Rate spent: held, started when the window frees (a fake clock, and a timer only while
    holding), with `queued` counted and not `submitted`.
  - Stale at start: a held run past `stale` ends `rejected reason=stale`.
  - Loop is never held.
  - A NATS redelivery of a rejected event gives no second `run.rejected`.
  - Held runs survive a restart (records only, a new supervisor), and their order holds.
  - Redispatch: a request file plus SIGHUP submits a run that skips stale, rate and loop. It is
    refused for a handled event without `--again`, for a disabled rule and for an `off` space; it
    is processed at start when anfsd was down; and the file is removed.
- **`dispatcher: none`:**
  - The space's rules show `off` and are not in the filters.
  - A matching event starts nothing.
  - Its queued and held runs end `canceled by=dispatcher:none` at reload; a working one goes on.
  - `match` says `off`.
- **idle CPU** with no runs: unchanged (under 0.2 s in 10 s). With one held run on a rate timer:
  the same.
- **e2e (`go/test/e2e.bats`, the throwaway nats-server):**
  - A `report.filed` line starts a run whose fake harness emits `report.closed` into
    `$ANFS_EVENTS_SPOOL`: through core's `anfs_emit` when `ANFS_CORE_LIB` is set (the job sets
    it to its core checkout's `lib/anfs`), else as a `printf`'d line of the profile.
  - The event is on `anfs.<sid>.report.closed` in the stream, with the stamped actor and
    `actorrun`.
  - A second fake run forges `actor` and arrives with `actorclaimed`.
  - A third fake run prints `env`, and the file has no `SSH_AUTH_SOCK`/`GH_TOKEN` though the
    e2e's anfsd had them.

core (bats):

- events.sh:
  - With `ANFS_EVENTS_SPOOL`, `anfs_emit` writes there and not to the log (both scopes, `--host`
    too).
  - The reserve: a 3700-byte line is refused there and accepted without the variable.
  - `ANFS_EVENTS=off` still writes nothing.
- `anfs agents`:
  - `redispatch` finds the line by id (a fixture log of two months), writes the request, signals
    a stub pid, asks without `-y`, and says "waits for anfsd" with no pid.
  - `ls --unhandled`; the `held:*` and `off` statuses from fixture records.
- `anfs bus status`: the held count.

anfs-dev: `bats packages/dev/tests`, unchanged (both rules still `disabled`).

## Verification

- While working: `anfs work run agents-02d runtime -- sh -c 'cd go && go test -race ./...'`
  (three times with `-count=3` at the end); the e2e bats with `ANFS_CORE_LIB` set; `anfs work run
  agents-02d core -- bats packages/anfs/tests/events.bats packages/anfsd/tests </dev/null`.
- Once, at the end: core's `packages/anfs/tests/all`, and anfs-dev's `bats packages/dev/tests`.
- No harness is started by any test or by the job. Nothing is installed, and the running anfsd is
  not touched.

## Done when

- A run's emits reach the bus with the run's stamped identity (`actor`, `actorspace`,
  `actorrole`, `actorrun`). A forged claim arrives visible in `actorclaimed`. `run.*` from a run
  is refused. Draining is proven at a normal end, in `finishDead`, at adoption and across a crash,
  with each id once in the files and on the bus.
- A run's environment is the allow-list. The enforcement table is in AGENTS.md, true for claude
  and stated as documented-only for the others.
- A dispatched claude loads no user or project settings, hooks, plugins, MCP servers, or the
  person's CLAUDE.md. `--bare` is one setting away for an API-key host.
- A full queue or a spent rate holds a run (bounded by `hold:`), and a held run survives a
  restart. Rejected-for-`queue`/`rate` events are unhandled and can be redispatched. Stale is
  checked when a run starts.
- `dispatcher: none` turns a space's dispatch off and changes nothing else.
- Every test above passes, core's suite included. The log lists what was not run.
- **After 02d is merged and installed, the person may turn on anfs-dev's two rules** (step 4
  below).

## By hand, for the person after merge

No migration code:

- Records from 02b keep their meaning. Their old `queue` or `rate` rejections become unhandled
  events, which `redispatch` can take.
- A run that was working across the upgrade has no spool and the old environment, and ends as
  before.

1. **Rebuild anfsd and restart it** (the release, `v0.3.0`, and core's pins can follow; this
   session can do that):

   ```bash
   cd ~/spaces/anfs-dev/runtime/go         # main, with agents-02d merged
   go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
   anfs bus restart anfsd
   anfs bus logs anfsd                      # rules: ... loaded, 2 disabled; consumers ready
   ```

2. **Core:** `anfs src update core && ppm install core/anfsd`, then `anfs bus status` (dispatch:
   0 held).
3. **A dry check:** `anfs agents match --type report.filed --space anfs-dev --subject
   r-20261024-99 --data to=file --data package=core/pcm --data title="a dry check"`. It shows the
   new command line (`--setting-sources local --strict-mcp-config`), the environment (no
   `SSH_AUTH_SOCK`, `ANFS_EVENTS_SPOOL` set) and `disabled: would submit`.
4. **Turn on anfs-dev's rules** (your decision; each report then costs a run, at most 10 USD): as
   02b's log step 5 (delete `enabled: false,` from both rules, commit, `anfs src update anfs-dev`).
   `anfs agents rules anfs-dev` then shows both `ok`. End the session that greps BACKLOG.md.
5. **The first real run is the live check** of what no test can do (02b's open checks included):
   - claude under launchd starts with the subscription login and loads `claude.json`;
     `anfs agents logs <run>`'s first line `[session ..., <model>]` means it started.
   - Its emits arrive stamped: `anfs events ls --type 'report.*' --json | tail -n 3` shows
     `actorrun` on any event the run wrote.
   - No herdr hook ran: herdr shows no pane for the run.
   - To check the environment at the cost of one small run: in a scratch space with an id, a
     rule with `job: none`, `budget_usd: 0.5` and the prompt "Run `env | sort` and `git config
     --get-all credential.helper`, and print both". Its summary must show no `SSH_AUTH_SOCK`,
     `GH_TOKEN` or `GITHUB_TOKEN`, and no helper. Remove the rule afterwards.
6. **If you would rather use strict isolation** with an API key: set `ANTHROPIC_API_KEY` in the
   service's environment (or an `apiKeyHelper`), `ANFSD_CLAUDE_BARE=on` in
   `~/.config/anfsd/anfsd.local.env`, then `anfs bus restart anfsd`. Runs are then billed to the
   API, not the subscription.

## Open questions (each with the recommendation)

1. **`--bare`.** It drops the subscription login (OAuth, keychain, `CLAUDE_CODE_OAUTH_TOKEN`).
   **Recommended:** the default is `--setting-sources local --strict-mcp-config` plus
   `claudeMdExcludes` (subscription auth, no user hooks, plugins, MCP or CLAUDE.md, but user
   skills and agents still visible); `--bare` is opt-in through `ANFSD_CLAUDE_BARE=on` with an API
   key. *Alternative:* `--bare` always, with a Console API key for anfsd (strictest, but billed
   per token). Another alternative, a separate `CLAUDE_CONFIG_DIR` for dispatched runs with a
   `claude setup-token` token, was considered and not taken: a second login to keep alive, and a
   long-lived token in the service's environment.
   **Decided 2026-10-24:** as recommended.
2. **A forged actor: keep and mark (`actorclaimed`), or reject the line?** **Recommended:** keep
   and mark. Honest tools with their own actor logic would otherwise lose work, and the stamp is
   the identity either way.
   **Decided 2026-10-24:** keep and mark.
3. **`run.delayed` or `run.submitted` with `held`?** **Recommended:** `submitted` with `held`, which
   stays inside A2A's states (`input-required` and `auth-required` are 04's, and those are the
   only additions planned).
   **Decided 2026-10-24:** `run.submitted` with `held` (and `detail`) in its data.
4. **`dispatcher: none` or `dispatch: off`?** **Recommended:** `dispatcher:`, a single-typed key
   that becomes the slot later.
   **Decided 2026-10-24:** `dispatcher:`.
5. **May a run emit into another space's scope?** The stamp says who did it. **Recommended:** yes
   until `accepts:` (cross-space is open, AGENT-NATIVE "Decided"). `run.*` is the only type
   refused.
   **Decided 2026-10-24:** yes, until `accepts:`.
6. **Defaults: `hold: 20`, `ANFSD_RUN_EVENTS_MAX=1000`.** **Recommended** as written. A burst of 30
   reports in an hour is an incident the person should see (`run.rejected ... hold full`), not a
   backlog that grows quietly.
   **Decided 2026-10-24:** as written.
7. **A role `env:` key** (more names passed through, e.g. `GOFLAGS`). **Recommended:** not now. The
   job's variables come from `anfs work env`, and nothing needs it yet. Add it when a rule does,
   with the credential names refused.
   **Decided 2026-10-24:** not now.
8. **Network for claude's sandbox** (`sandbox.network.allowedDomains`, e.g. `proxy.golang.org`).
   **Recommended:** none in 02d. "No network" is what makes push impossible; a per-space
   `network:` list (never `github.com`) is a later plan when a fix needs downloads.
   **Decided 2026-10-24:** none in 02d.

## Out of scope (later)

- F8: per-scope shipping isolation, the sink's stall, `DeliverNewPolicy` gaps, and retention of
  `runs/` (run spools included) and the month files.
- F2 and F3 (an open envelope, direct NATS publishing and a recorder, a tool NATS user), and 05's
  registration of `actorrun` and `actorclaimed`.
- The general `tools:` slots schema (`dispatcher:` is its first key, nothing more); harness
  definitions as data (package YAML); the event catalog (`anfs events types`).
- `anfs inbox` listing unhandled events (04).
- **Follow-up, named: runs file reports by event.** A run files a report by emitting
  `report.filed` into its own spool (section 1); anfsd ingests and stamps it like any run event,
  and a consumer records it (the review's projection, 4.2), replacing `anfs report new`'s write of
  the record in STATE, which is outside the sandbox, when it runs inside a run. Better than
  opening STATE to runs. In `BACKLOG.md`.
- Isolation for antigravity, and opencode's global config, beyond what section 3 finds.
- Signing or attesting actors for spool writers that are not runs (F6: not on a dev host).
