---
plan: agents/02b-dispatch
job: agents-02b
date: 2026-10-23
---

# Log: agents/02b, dispatch, supervision, `access:`, and the report as a view

## What was built

- **anfs-io/runtime** (anfsd): `internal/space` (`space.yml`'s `access:`, `agents:`, `dispatch:`:
  the one parser, every key validated, an invalid rule refused with its reason while the space's
  others load; templates; matching; the consumer's filter subjects; `Discover` over `wsm ls
  --json`); `internal/dispatch` (the durable consumer `anfsd-dispatch`, filters each rule's
  `anfs.<space id>.<type prefix>` plus `anfs.host.source.updated`, rules read again on SIGHUP and on
  `source.updated`; the guards `stale`, `rate`, `queue`, `loop` as `run.rejected`; one run per rule
  and event id; `Decide` for the dry check); `internal/supervise` (records in
  `~/.local/state/anfsd/runs/<run id>/`, a queue, slots per role and `ANFSD_MAX_RUNS`, `anfs work
  new|path|env`, setsid, timeout SIGTERM then SIGKILL to the group, the `cancel` marker, adoption
  after a restart, `orphaned`, retries with a backoff, `run.*` through the spool);
  `internal/harness` (one file per harness: argv, environment, claude's settings file, the summary,
  the text of a stream); `internal/sink` (`anfsd-sink-report`: `anfs report __sink` per `report.*`
  event, one at a time, 5 tries); `bus.Consume`/`UpdateConsumer`; `anfsd rules`, `anfsd match`,
  `anfsd render`; `internal/testfake` (the fakes every test runs instead of programs).
- **core**: core/anfsd's **`anfs agents`** (`ls`, `show`, `logs`, `stop`, `rules`, `match`);
  `anfs bus status`'s `dispatch` row and a `harness <name>` row per harness a rule uses; `anfs bus
  reload` documented as reading the rules; `ANFSD_MAX_RUNS` in anfsd.env; `pins.yml` names v0.2.0.
  `lib/anfs/report.sh`: a file report's **record** in `$ANFS_STATE_HOME/reports/<space id>/<id>.md`,
  ids unique per space (made with `set -C`), `status`/`close`/`wait` on the record, **`anfs report
  show`**, **`anfs report __sink`** (the render: `report.filed` appends once, `report.closed`
  rewrites the `Status:` line, under a `mkdir` lock beside the file), `new` and `close` rendering
  themselves while anfsd is down or events are off. `cli.sh`: a group's `__<name>` plumbing runs.
  `events.sh`: `run.*` in the completion types. `anfs rpc`: `rules` and `match` are reads.
  AGENTS.md ("Dispatch", new; "Reporting a bug"; `space.yml`'s keys; "Events"; "The bus"; the
  lists; the XDG table; "Testing"), README, the `anfs-bug-report` skill, anfsd's README.
- **anfs-dev**: `spaces/space.yml` (`access:`, `agents: maintainer`, the rules `fix-core-report`
  and `fix-stack-report`, **both `enabled: false`**), `spaces/prompts/report-filed.md`,
  `packages/dev/tests/space.bats`.
- **docs**: the plan's harness table corrected (below), ROADMAP's "Retiring" line, this log.

## Commits (branch agents-02b, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| runtime | `44c7a72` | dispatch, supervision, harnesses, the sink, `rules`/`match`/`render`, tests, README, AGENTS.md |
| core | `25492c2` | `anfs agents`, bus status rows, the report records and `__sink`, docs, tests (anfs stays 0.2.17: bumped once since its last push; anfsd 0.2.0) |
| anfs-dev | `270bce5` | the two rules (disabled), the prompt, `space.bats` |
| docs | (this) | the plan's table corrected, ROADMAP, this log |

## The job's first task: the claude and agy rows

Checked with `--help` (and `--version`) only, as the person asked: no harness was started, so the
plan's `-p "say ok"` checks were not run.

- **claude 2.1.296** (`claude --help`, and its binary's strings for the settings keys): `-p` is a
  flag that takes no value and `--add-dir <directories...>` takes several, so the plan's `claude -p
  <prompt> ... --add-dir <p>` would let `--add-dir` swallow what follows: the prompt now goes last,
  after `--`. `--permission-mode dontAsk`, `--settings <file-or-json>`, `--max-budget-usd` (with
  `--print` only), `--model`, `--name`, `--output-format stream-json`, `--verbose` all exist. The
  sandbox keys exist: `sandbox.filesystem.allowWrite`, `sandbox.failIfUnavailable` (set true: a
  missing backend fails instead of running the shell unconfined), `sandbox.allowUnsandboxedCommands`
  (set false). The `:*` prefix syntax of `Bash(...)` rules is still supported. **Finding: under
  `-p`, "settings files that fail validation are silently ignored"** (its `--help`): anfsd writes
  `claude.json` from a struct and a test holds its shape (no unknown key, each value), but whether
  2.1.296 accepts it was not run here. The person's first run (by hand step 5) is that check: `anfs
  agents logs <run>` shows a refused tool if the permissions loaded. Also new in 2.1.296:
  `--permission-prompts none`, `--restricted` (file tools confined to the working directories,
  user and project settings ignored, Bash removed unless `--tools` names it), `--setting-sources`;
  none used (open questions).
- **agy 1.3.3**: `agy --help`, `agy -h` and `agy --version` **all hung**, no output, 10 s each with
  stdin `/dev/null` (killed by an alarm; nothing left running). The flags (`-p`/`--prompt`,
  `--sandbox`, `--add-dir`, `--print-timeout`) are confirmed only in its binary's strings and
  changelog, which also says an expired `--print-timeout` exits 0 with the partial answer: anfsd's
  own timeout (the same length) is what fails the run. Whether `agy -p` runs headless without
  hanging is unverified; its rules stay `partial`, and `anfs bus status` never asks its version.
- opencode 1.18.35 and pi 1.1.0: their `--help` matches the plan's rows (opencode shows no `--`,
  so its prompt is the plain positional; pi documents `--`).

The rows in `02b-dispatch.md` are corrected to this.

## Decisions made during the work (deviations from the plan, and why)

1. **`enabled: false`** is a rule key (default true; status `disabled`: valid, shown, matched by
   the dry check, never run). The plan had no switch, and the person required the anfs-dev rules
   committed disabled behind one.
2. **The dry check, `anfs agents match` / `anfsd match`** (new): an event from a file or stdin, or
   built from `--type --subject --data --space`; per rule: no match (and why: `data.package:
   stack/go is not core/*`), submit (the job, the cwd, the environment, the harness's command line
   with the prompt cut), reject (why), duplicate, or disabled (what it would do). It reads the run
   records for the guards and writes and starts nothing.
3. **`args:`** (a rule's extra harness options): the plan refuses "a skip-permissions harness
   option" but the format had no place for one; `args:` is it, and refuses skip-permission flags
   and the ones that would override what anfsd hands the harness (`--permission-mode`,
   `--settings`, `--add-dir`, `--sandbox`, `--dir`, `--tools`, ...).
4. **The next id counts the file's `## r-<day>-<n>` headings too**, not only the records: else the
   first report of a day that already has text-only sections would reuse an id and the sink would
   skip it (a heading is there). A read, not a conversion.
5. **`report.closed` for a record whose section is missing from the file appends the record** (an
   idempotent repair) instead of doing nothing; `__sink` renders only `file` reports whose record
   is in the file's own space (a file is one space's view).
6. **A `/bin/sh` wrapper writes the harness's exit status** into the run's directory, so a run that
   ends while anfsd is down, or while adopted (anfsd is not its parent), ends with its real status;
   `orphaned` is only a run that left neither an exit status nor a cancel marker.
7. **opencode rules are `partial`** too (plan.md, decision 5: "Partly"; 02b named only
   antigravity).
8. **claude's settings** also set `defaultMode: dontAsk`, `sandbox.failIfUnavailable: true`,
   `sandbox.allowUnsandboxedCommands: false`, and deny `Bash(<cmd>)` beside `Bash(<cmd>:*)`.
9. **Retries back off** (30 s, then 2 min), so a failing harness never restarts at once; a run is
   otherwise never restarted, and `rate:` (default 20/h) caps starts per rule. Rejected runs have
   records too (a redelivery is acked without a second `run.rejected`); `rate:` counts submitted
   runs only.
10. **`cli.sh`**: in a group, an undeclared `__<name>` sub-command runs its `cmd_` function (only
    `__` names), which `anfs report __sink` needs.
11. **`anfs agents stop`** signals a group only when the recorded process's start time still
    matches (a reused pid is not signaled); a queued run gets the marker and is canceled when the
    scheduler next looks (a run starting or ending), not at once.
12. **`anfs bus status`**: an invalid rule makes the `dispatch` row `fail` (exit 1), saying
    `anfs agents rules`; a harness row asks `<harness> --version`, never agy's.
13. **`pins.yml` names v0.2.0**: no v0.1.0 tag exists, so the first release is the plan's v0.2.0.
14. **The 02a e2e tests run anfsd with `PATH=/usr/bin:/bin`**: anfsd now also runs `wsm`, `anfs`
    and harnesses, and must never find the person's in a test. The new e2e test puts the fakes
    first. The pinned nats-server and nats CLI were linked from `~/.local/share/nats/` into the
    job's TMPDIR.
15. A `job: none` run's TMPDIR is `<run dir>/tmp`.

## Tests

- **Go** (`go test -race ./...`, three runs with `-count=3`, none flaky; gofmt and vet clean): **67
  tests** (02a: 29). New: space (every key, 31 invalid cases, missing programs, statuses,
  duplicates, matching, filter subjects, templates, discover); harness (golden files per harness,
  claude.json's shape, opencode's config, pi read-only, summaries, render); supervise (a job made
  with the rendered name and repos, the argv, the environment: `ANFS_ACTOR`, `ANFS_AS`, `ANFS_RUN`,
  `ANFS_EVENT`, the job's variables, no `ANFS_WORK`, stdin empty; `job: none` with each other
  harness; a retry reuses the job; a foreign job fails; spawn failure; concurrency per role and
  host; a timeout kills the fake and its child; the cancel marker; a restart adopts a live run,
  orphans a dead one, starts a queued one; an adopted run's timeout; every transition one valid
  line); dispatch (filters, one run per event, stale, queue, loop, rate, the dry decision, the
  consumer on JetStream with a `source.updated` reload); sink (each `report.*` on stdin in order,
  a failure retried, given up after 5); cmd (rules, match, render, **anfsd run dispatching end to
  end** with the fakes). **Idle anfsd: 44 to 65 ms of CPU over 11 s** with the dispatcher and the
  sink consuming (the limit is 200 ms in 10 s; 02a's was 32 ms).
- **e2e** (`go/test/e2e.bats`, the pinned nats-server v2.15.1 and nats v0.5.0): **4 passing**
  (02a's 3, and a `report.filed` line in the spool giving `run.submitted`, `run.working`,
  `run.completed` in the stream, the fake harness run in its job, `commits` `core:1`).
- **core** (`packages/anfs/tests/all`, once at the end): **740 passed, 0 failed**, 54 files (pmm and
  pcm off), 58 s (02a ended at 726). New: report.bats (+5: the record and its view, ids unique per
  space with two sink files and a pre-record heading, ten `new` at once, `__sink` once however
  often and the `Status:` rewrite, the lock), agents.bats (8, new: ls, show, logs, stop on a real
  process group incl. SIGKILL and a reused pid, rules, match, the seam), bus.bats (+1: the dispatch
  and harness rows).
- **anfs-dev** (`bats packages/dev/tests`): **41 passed** (+3, space.bats: both rules valid and
  disabled, `ok` once enabled, match: a core report a core job, a stack report a stack job, no
  package nothing).
- One incident: an early run of the new bus.bats test had the person's PATH and found the real
  `pi`, so `anfs bus status` ran `pi --version` once (a version query, which the brief allows; the
  rule is that tests stub every harness). bus.bats and agents.bats now run with a PATH of their
  stubs, yq, bash and the system's directories only.

## Not run

- No harness was started (claude, opencode, pi, agy), not even `-p "say ok"`: their flags are from
  `--help` (above). The real `claude.json` loading under `-p` is unverified.
- Nothing on this host was installed, loaded or restarted; the running bus was not touched; every
  NATS was an embedded one or a throwaway nats-server on a random port in a temp dir.
- pmm's and pcm's suites (off); CI and the release (nothing pushed); Linux (the supervisor's
  `ps -o lstart=` and setsid are portable, but untested there).

## By hand, for the person after merge

1. **Rebuild anfsd from source and restart it** (the release, v0.2.0, can follow: push runtime, `git
   tag v0.2.0 && git push origin v0.2.0`, then the sums into core's `packages/anfsd/pins.yml` with
   `packages/anfsd/tests/pins --print`; this session can do that for you):

   ```bash
   cd ~/spaces/anfs-dev/runtime/go          # main, with agents-02b merged
   go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
   anfs bus restart anfsd
   anfs bus logs anfsd                      # "rules: ... loaded", "consumer anfsd-dispatch: filters ...",
                                            # then both consumers "ready"
   ```

   If anfs-dev's merged `space.yml` is already in its source, its two rules show `invalid`
   ("prompt_file prompts/report-filed.md: no such file") until step 2's `wsm install anfs-dev`
   links the prompt.

   The consumers are made deliver-new: nothing that happened before is acted on. Do step 2 at
   once: until core is updated, a report event's render fails (the installed `anfs` has no
   `report __sink`) and is retried 5 times over about 8 minutes, then dropped (the next `new` or
   `close` renders it again).

2. **Reinstall the packages** (the new `anfs agents`; the prompt file's link in the space):

   ```bash
   anfs src update core anfs-dev            # lib/anfs is a directory link: live at once
   ppm install core/anfsd                   # links anfs-agents; rewrites and reloads the unit (ANFSD_BIN)
   wsm install anfs-dev                     # links spaces/prompts/report-filed.md into ~/spaces/anfs-dev
   anfs bus reload
   anfs bus status                          # dispatch ok: "0 rules loaded, 2 disabled, 0 invalid; 0 working, 0 queued"
   ```

   (Never `ppm install -r core/anfs`.)

3. **The existing BACKLOG.md** (`~/spaces/anfs-dev/docs/BACKLOG.md`, `ANFS_REPORT_TO`): nothing
   converts it.
   - Its 8 report sections (`r-20261022-1` to `-7`, `r-20261023-1`) have no record: they stay as
     text. The two still open, **`r-20261022-4`** and **`r-20261022-7`**, are closed by editing
     their `Status:` line by hand when fixed (`Status: fixed in <source> <sha> (<date>)`):
     `anfs report status|show|close` on them now exit 2 ("filed before the records ... text
     only"), and `anfs report wait` cannot follow them.
   - New reports are recorded in `~/.local/state/anfs/reports/tkj76y3nfx/` and appended to the
     same file by anfsd; their ids skip the headings already there.
   - Delete the bullet near the end of the file (after `r-20261023-1`'s diagnostics) that begins
     "`anfs report` ids (`r-YYYYMMDD-N`) are counted per sink file": ids are unique per space now.
     The other bullets there are notes, not reports: new reports are appended after them.
   - From now on never edit a report's section that has a record: the render rewrites it. Close
     with `anfs report close <id> --commit <sha>`.

4. **A dry check of the rule**, which starts and writes nothing:

   ```bash
   anfs agents rules anfs-dev               # fix-core-report, fix-stack-report: disabled
   anfs agents match --type report.filed --space anfs-dev --subject r-20261023-99 \
     --data to=file --data package=core/pcm --data title="a dry check"
   ```

   It prints, for `fix-core-report`, "disabled: would submit, but the rule has enabled: false",
   the job (`anfs work new fix-r-20261023-99 core`), the cwd, the environment and claude's whole
   command line; for `fix-stack-report`, "no match: data.package: core/pcm is not stack/*". A real
   past event works too: `anfs events ls --type report.filed --json | tail -n 1 | anfs agents
   match --event -`.

5. **Turn anfs-dev's rules on** (your decision; each report into anfs-dev then costs a run, at most
   10 USD): delete `enabled: false,` from both rules in `spaces/space.yml` of the anfs-dev source,
   commit, and reload:

   ```bash
   cd ~/.local/share/anfs/sources/anfs-dev  # or in a job: anfs work new dispatch-on anfs-dev
   sed -i '' 's/ enabled: false,//' spaces/space.yml
   git commit -qam "anfs-dev: dispatch on"
   anfs bus reload                          # (an anfs src update after a merge reloads by itself)
   anfs agents rules anfs-dev               # both ok
   anfs bus status                          # dispatch: 2 rules loaded; harness claude ok
   ```

   Then end the session that greps BACKLOG.md. The first real report into anfs-dev is also the
   check that claude 2.1.296 runs under launchd (logged in, found on the service's PATH) and loads
   the run's settings file: watch it with step 6.

6. **The runs**:

   ```bash
   anfs agents ls                           # RUN STATUS RULE ROLE SPACE HARNESS JOB AGE (--all: every one)
   anfs agents show <run>                   # the record, the event, the command, the job
   anfs agents logs <run> -f                # claude's stream as text
   anfs agents stop <run>                   # asks; -y does not. SIGTERM to its group, SIGKILL after 30 s
   anfs agents stop --all -y
   anfs events ls --type 'run.*'            # the run.* events
   ```

   A run is named by its id or the id's last 8 characters. A stopped run's job stays; review it
   (`anfs work show <job>`), merge or `anfs work rm <job>`, then `anfs report close <id> --commit
   <sha>`.

## Open questions

1. Claude Code still loads the person's user settings and a job's project settings beside the
   run's `--settings`; their deny rules only narrow, but an allow rule there (`Edit(~/**)`) widens
   what a dispatched agent may write. `--setting-sources` (or 2.1.296's `--restricted`, which also
   removes Bash unless `--tools` names it) would shut that; neither is in the plan. Worth deciding
   before 04.
2. Should an invalid rule fail `anfs bus status` (exit 1)? It does now, like a lag.
3. A host whose `~/.local/state/anfs/events/` does not exist when anfsd starts ships its first
   events only at the 30 s reconcile (02a's watcher; found by a test here). Not this host.
4. agy's headless behaviour is unverified (its help hangs); keep antigravity off anfs-dev's rules.
