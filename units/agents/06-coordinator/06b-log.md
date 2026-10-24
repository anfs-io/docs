---
plan: agents/06-coordinator (06b)
job: agents-06b
date: 2026-10-24
---

# Log: agents/06b, the Claude adapter, settings ownership, the inbox line

## First task: `asyncRewake` and the hook timeout

Checked against https://code.claude.com/docs/en/hooks.md (fetched 2026-10-24, the whole page in
three parts) and the installed Claude Code 2.1.296. The binary was only read with `strings`.
`claude` was never run. The plan's "a fake hook in a scratch project" needs a real session, and
this job's rules forbid one.

What the page says:

- `timeout`: "Seconds before canceling". Defaults: 600 for `command`, 30 on `UserPromptSubmit`.
- "Once an async hook is running in the background, Claude Code doesn't enforce `timeout` on it.
  Claude Code still enforces `timeout` on a hook you run with `asyncRewake`."
- "an `asyncRewake` hook that exits with code 2 wakes Claude immediately even when the session
  is idle". The hook's stderr (or its stdout, when stderr is empty) is shown to Claude as a
  system reminder.
- In `-p` mode, async hooks still running at teardown are killed. The page says nothing about
  interactive sessions.
- The only maximum anywhere is SessionEnd's: a 1.5 s budget, raised by a per-hook `timeout` up to
  60 s.

What the binary shows (2.1.296):

- The handler schema is `timeout: number().positive().optional()`, with no upper bound.
- `asyncRewake` is described as "Implies async". It is a field of any command handler, whatever
  the event, so SessionStart and Stop both take it.
- The timeout is `timeout * 1000` ms, passed to a plain `setTimeout`.
  - 86400 s is 86,400,000 ms. A JS timer accepts up to 2^31−1 ms (about 24.8 days).
  - So a day is accepted and enforced as written.
- `asyncRewake` runs in the background when the session is interactive (or has streaming input).
  In plain `-p` it runs in the foreground. That does not matter here: a dispatched run loads no
  settings at all.
- At shutdown, Claude Code kills its live shell commands that are still "running". Whether a
  backgrounded hook counts as one is not certain. It does not matter either: the listener ends
  within a second of its holder's death (tested).

**Decision: hooks (`arm: hooks`), with T = 86400 s.** That is well above the plan's one-hour
threshold. The hook runs `wsm listen --hook --max 86340s`, so an idle day costs one small wake
("no events ... still listening"). The fallback (`coordinator: {arm: task}`) is built and tested
too.

Not confirmed, because it needs a real session:

- that an interactive session's async hooks die with it;
- that SessionStart's `asyncRewake` entry behaves exactly like Stop's.

The person's first session is that check.

## What was built

**runtime** (anfsd):

- `internal/land/summary.go`: `Summary` keeps `~/.local/state/anfsd/inbox/summary.json`. It is one
  compact line:

  ```
  {"v":1,"total":N,"spaces":{"<space id>":{"space","approvals","unhandled","oldest","items":[two]}}}
  ```

  - Items are `<step> <what>`, as `anfs inbox ls` shows STEP and WHAT.
  - Unhandled work is found by the same rule as core's `_ib_unhandled`.
  - A request with no space is under `host`.
  - It is rewritten (temp file, then rename) 500 ms after any approval or run record is saved,
    and after each `Process` (a dismissal renames a file and saves no record). It is only
    rewritten when it changed. Nothing polls.
- `supervise.Store` and `land.Store` gain `OnSave`. `Lander.Summary` and
  `config.Paths.InboxSummary` are new.
- `cmd/anfsd` wires the summary up.
- `dispatch_test`'s space now has `coordinator:`, and match still shows `--setting-sources=`.
- README and AGENTS.md.

**core**:

- `lib/anfs/claude/hook`, the adapter. It has five entries:
  - `session-start`
  - `listen [--max S]`
  - `prompt [--arm task]`
  - `inbox`
  - `session-end`

  How it works:
  - It reads the hook's JSON with bash regexes: no jq, and no wsm or yq on the prompt path.
  - The space is the one at or above the input's `cwd`.
  - The holder is the input's `session_id` plus the Claude Code ancestor: `comm` is `claude`, or a
    node running `claude-code/cli.js`, within six levels.
  - It exports `CLAUDECODE=1` and `CLAUDE_CODE_SESSION_ID`, so the lease events' actor is
    `claude:<session 8>`.
  - Per-session files are `$ANFS_STATE_HOME/claude/<session>.{lease,inbox}`.
  - It measured about 23 ms per prompt as the holder, and 14 ms for the inbox entry alone (macOS).
- `wsm` (0.13.0, package 0.1.13; both already bumped in this unpushed series):
  - `coordinator:` is parsed (`_coord_parse`, read with one yq as JSON).
  - `.claude/settings.json` is merged on install. The hooks come out on remove, implode and for a
    vanished space.
  - `wsm ls` reports the state (NOTE, and `--json` `claude`).
  - `wsm add` skips the file.
  - `wsm listen` takes the coordinator's wake set and window from `coordinator:`.
- The wsm test helper:
  - sets `ANFSD_STATE_HOME` and `ANFSD_CONFIG_HOME` into the test's temp dir, and the guard
    checks them;
  - unsets `ANFS_COORDINATOR` and `ANFS_INBOX_HINT`.
- `settings.bats` knows `ANFS_COORDINATOR`, `ANFS_INBOX_HINT` and `ANFS_RUN` are not settings.
- AGENTS.md:
  - "`space.yml`'s `coordinator:`" and "Claude Code settings" (what wsm owns);
  - under "Leases": the adapter and the inbox line;
  - the tests.
- The wsm README.

**anfs-dev**:

- `spaces/space.yml`: `coordinator: claude` (the default wake set: the plan names no other), with
  a comment.
- `space.bats` gains a test: `coordinator: claude` is read, and a dispatched claude's argv keeps
  `--setting-sources=`.

**docs**: this log.

## Commits (branch agents-06b, not pushed, not tagged)

| Repo | Commit | What |
| --- | --- | --- |
| runtime | `df888de` | the inbox summary (`land.Summary`, the stores' `OnSave`, `Paths.InboxSummary`, wiring), tests |
| runtime | `dc510d3` | dispatch_test: `coordinator:` and `--setting-sources=`; README, AGENTS.md |
| core | `aec4f06` | wsm: `coordinator:`, `.claude/settings.json` (merge, ls, remove, add), listen's wake set; helper; claude-settings.bats; README |
| core | `27532aa` | `lib/anfs/claude/hook`; claude-hook.bats; settings.bats |
| core | `d179042` | AGENTS.md |
| anfs-dev | `005186c` | `coordinator: claude`; space.bats |
| docs | (this) | this log |

## Decisions made during the work (deviations from the plan, and why)

1. **Settings are merged, not owned by sha.**
   - This follows the job's direction: wsm writes only its own hooks and keeps everything else
     the person put there. Plan section 6 had the whole file as wsm's, with a sha in
     `$WSM_STATE_HOME/claude/<id>.sha256` and "changed by hand: refuse". Neither exists.
   - wsm's hooks are the handlers whose `command` ends in `/anfs/claude/hook`, so an old path from
     another HOME is also taken out.
   - Each install removes them all and appends what `coordinator:` asks for, at the end of each
     event's list. Groups and events left empty are dropped.
   - Only a file that is not a JSON object is refused (`-f` moves it aside). A link is refused:
     one owner.
2. **`wsm ls`** says `claude settings outdated|invalid|refused` in NOTE (STATUS `partial`), and
   `--json` gains `claude`. There is no new column; the table keeps its four.
3. **An `inbox` entry** for spaces without a coordinator. The plan had `prompt` everywhere. With
   `inbox`, a `coordinator: none` space never walks the process tree.
4. **`listen --max`** comes from wsm (`args: ["listen","--max","86340"]`), so T is written in one
   place.
5. **`CLAUDE_PID` is never trusted**, not even when it names the ancestor. The walk is the only
   source; it is just as safe and simpler.
6. **The `listen` hook exits 0, not 1**, when the listener ends without a wake (another listener,
   lost claim, holder gone, the hook's SIGTERM). Claude Code then shows nothing. Exit 2 is the
   only wake.
7. **SessionStart's `listen` claims too**, in parallel with `session-start` (as the plan says).
   - Stop's `listen` only re-arms the holder. If the holder's own lease expired (a turn longer
     than `work_ttl`), Stop claims it again.
   - The startup brief says "is", even when the `listen` entry claimed a moment before. "Is still"
     is for `compact` and `resume`.
8. **The summary's layout**: a top-level `total` (so "elsewhere" is `total − here`), and items
   prefixed with the step. It is rewritten only when it changes.
9. **"Unhandled" exists twice**: once in Go (the summary) and once in core's bash (`anfs inbox
   ls`). It is the same rule; see open question 1.
10. **`wsm add` never moves `.claude/settings.json` into the source.** Without this, `add .` would
    have taken wsm's own file (found by manage.bats).
11. **`remove` takes the hooks out just before `space.yml`**, so an interrupted remove leaves the
    space as it was (manage.bats's "interrupted remove" test).
12. **Versions**:
    - `WSM_VERSION` stays 0.13.0. 06a set it in this same unpushed series, and the convention is
      one bump per series.
    - The pre-commit hook left anfs 0.2.19, wsm 0.1.13 and dev 0.1.18, all already bumped since
      the last push.
    - Runtime has no version to bump (a tag is the release).
13. **The fake Claude Code in the hook tests** is a copy of bash named `claude` running a driver
    that reads requests from a FIFO. One session's hooks are then children of one long-lived pid,
    as Claude Code's are. That is what the "same holder", takeover and listener-death tests need.

## Tests

Every run was `anfs work run agents-06b <repo> -- <iso> <cmd> </dev/null`.

`<iso>` starts the command under `env -i` with:
- a fresh HOME, every `XDG_*` (`XDG_RUNTIME_DIR` too), `TMPDIR` and `GOCACHE`, all under the job's
  TMPDIR (`/tmp/anfs-work-501-1/iso.*`);
- PATH: the system's directories plus links to go, gofmt, bats, yq, bash, git, jq, nats-server
  v2.15.1 and nats;
- `GOPROXY=off`, and the module cache read-only.

The anfs/wsm homes are left to derive from HOME and XDG. A first full run that set them
explicitly broke tests that set only HOME (push, onboard, stacks, system, ppm confirm).
`ANFS_WORK` was never set.

- **runtime**:
  - `go test -race ./...`: every package ok (gofmt and vet clean), apart from
    `supervise.TestRunSpoolRefusals`, which failed once. It is a pre-existing millisecond race:
    `spoolLine` takes `time` and the id's time from two clock reads, so "time is not the id's"
    when they straddle a millisecond. It passed on `-count=3` of supervise, land and cmd/anfsd.
  - New tests: `TestComputeSummary`, `TestSummaryFollowsTheRecords` (merge, install, dismissal,
    a held run starting, atomic reads, no rewrite without a change, mode 0600),
    `TestSummaryExpiry`.
  - e2e (`go/test/e2e.bats`, `ANFS_CORE_LIB` the job's core): **7 passing**.
- **core**:
  - New: `wsm/tests/claude-settings.bats` **11**, `anfs/tests/claude-hook.bats` **11**.
  - `packages/wsm/tests` together with claude-hook, events and files: **216 passing**.
  - `packages/anfs/tests/all` (pmm and pcm off): **833 passed, 0 failed**, 60 files, 125 s.
  - What claude-hook.bats covers:
    - claim and brief; a second session named; compact re-briefs with no event; the holder's
      prompt is silent;
    - a dead holder's lease taken (`lease.expired why=dead`, `lease.claimed from=dead`);
    - an expired one taken on the fake clock; the lost-lease line once; release with `clear` and
      `session-end`; `from=released`;
    - no claim outside a space, under `ANFS_COORDINATOR=off`, with `ANFS_RUN` set, for an
      `agent_id`, with no Claude ancestor (`CLAUDE_PID` set), or with no input;
    - the `sh -c` wrapper chain; `CLAUDE_PID` naming another process ignored;
    - the parallel SessionStart pair gives one claim; an event wakes (exit 2, the batch on stderr,
      stdout empty); Stop re-arms with the batch committed; a newer listener ends the older with
      0;
    - a non-holder's listen exits 0 at once; `--max` gives one line; SIGTERM ends the listener
      (lease back to `working`); the holder's death ends both the hook and its listener;
    - idle: under 0.2 s of CPU in 5 s for the waiting hook and its listener;
    - `arm: task`: the brief and the "listener is not running" line;
    - the inbox line: on change, after 30 min, `+N more`, elsewhere only, `ANFS_INBOX_HINT=off`,
      nothing when nothing waits or there is no file;
    - the `inbox` entry in a `coordinator: none` space; the session's files go at SessionEnd.
  - What claude-settings.bats covers:
    - the inbox entry and the five entries (the absolute path, exec form, `asyncRewake`, the
      timeouts); `arm: task`;
    - six invalid `coordinator:` forms warn and count as none;
    - a person's keys, hooks and `settings.local.json` kept; idempotent (same bytes); an old-path
      entry replaced; claude to none;
    - remove keeps the person's file, and deletes wsm's alone with `.claude/`;
    - not JSON refused, `-f` moves it aside; a JSON array refused; a source-shipped file refused;
    - a dry run writes nothing; `$HOME/.claude` never touched;
    - `wsm listen` wakes on `coordinator: wake` with its window.
- **anfs-dev**: `packages/dev/tests` **43 passing** (space.bats 4, built from this job's runtime).

## Not run

- A real Claude Code session (by the rules). See the first task for what only it can confirm.
- Linux.
  - The hook avoids GNU/BSD differences: `ps -o ppid= -o comm=`, the comm's basename, `find
    -mtime -delete`, no `date`.
  - The CPU test reads `/proc/<pid>/stat` there.
  - The fake claude is a copy of bash, so `comm` is `claude` on both.
- Nothing installed. anfsd, NATS and the ui were not touched. No `wsm lease` or `listen` against
  real state. No real space's `.claude/` was read or written.

## By hand, for the person (no migration code)

04a and 06a merged and installed first.

1. **Merge**: ff-merge `agents-06b` in core, runtime, anfs-dev and docs. Rebase first if main moved.
   anfs-dev's commit touches `spaces/space.yml`, which is yours.
2. **anfsd** (the summary): rebuild from source and restart.

   ```bash
   cd ~/spaces/anfs-dev/runtime/go && go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
   ```

   Then `anfs bus restart anfsd`. `~/.local/state/anfsd/inbox/summary.json` appears at start.
3. **Install**:
   1. `anfs src update core anfs-dev`.
   2. `ppm install core/anfs core/wsm` (the hook, wsm). core/anfsd only if 04a's is not in yet.
   3. `wsm install anfs-dev`. It prints `claude settings: written (the coordinator (hooks) and the
      inbox line)`.
   4. Check `wsm ls anfs-dev --json`: it says `"claude":"ok"`. Look at
      `~/spaces/anfs-dev/.claude/settings.json`: anything you had there is still there.
4. **Retire the hand-run listener**: end the background `anfs events ls -f ... | grep -m1 ...` task
   in today's coordinator session, and drop the instruction to re-arm it from its notes or memory.
5. **Try it**:
   1. Start Claude Code in `~/spaces/anfs-dev` (`wsm cd anfs-dev; claude`). Accept the workspace
      trust dialog if asked: hooks wait for it.
   2. It should have the brief in context ("this session is the coordinator of anfs-dev").
      `wsm lease holder coordinator` names it, `listening` within a second or two.
   3. Start a second session there. It is told who the coordinator is.
   4. File a test report from cnfs. The first session wakes within about 6 s (the 5 s window) with
      the batch.
   5. Quit the first and type in the second. It says it took over (`released`, or `dead` if
      killed), and an event filed in between is in its first batch.
   6. With an approval waiting, any session's next prompt carries the `anfs inbox: ...` line.
   7. If a wake never comes, `anfs log` and `--debug` (`Hooks:` lines) show what the listen hook
      did. Then `coordinator: {harness: claude, arm: task}` in space.yml plus `wsm install
      anfs-dev` is the fallback.
6. **Other spaces** get the inbox line at their next `wsm install`. To give a space a
   coordinator: `coordinator: claude` in its space.yml (your commit), then `wsm install <space>`.

## Open questions

1. **"Unhandled work" is computed twice**: Go (the summary) and bash (`anfs inbox ls`).
   **Recommendation:** for now, a shared fixture test (the same records, both answers). Later,
   `anfs inbox ls` could read anfsd's view when anfsd is up.
2. **Every turn's Stop starts a new hook and `wsm listen`** (bash and the libraries: about
   0.1–0.2 s of CPU per turn), and the previous listener ends. **Recommendation: accept.** If it
   shows, the Stop entry could exit at once when its session's listener is alive and registered
   (one `kill -0`).
3. **Two things are unverified until a real session**: interactive teardown of async hooks, and
   SessionStart's `asyncRewake`. **Recommendation:** the person's first session (by-hand step 5).
   `arm: task` stays as the documented fallback.
4. **`supervise.TestRunSpoolRefusals` is flaky** (pre-existing; the millisecond race above).
   **Recommendation:** a one-line fix in its `spoolLine`: derive `time` from the id's
   milliseconds.
5. **Sessions started below a space's root** get no hooks (plan open question 3). **Recommendation:
   accept, as agreed.** `wsm cd` lands at the root.
