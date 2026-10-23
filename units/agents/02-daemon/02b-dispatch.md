# Plan 02b — dispatch, supervision, `access:`, and the report as a view

Written 2026-10-23, to be built after 02a is merged and installed. Part of [plan 02](plan.md),
whose decisions 3, 4 and 5 this builds. Mergeable alone: after it, a report filed into anfs-dev
starts the maintainer in a job with nobody watching, and BACKLOG.md is written by the bus.

## Context — read these files first

The job is `anfs work new agents-02b core runtime anfs-dev docs`.

- docs: `AGENT-NATIVE.md`, [plan.md](plan.md), [02a](02a-bus.md) and its log,
  `units/report/01-anfs-report/plan.md` (the file sink, ids, `close`, `status`)
- core `AGENTS.md`: "wsm: spaces" (`space.yml`), "Reporting a bug", "Events", "The bus" (02a);
  `lib/anfs/report.sh`; `packages/anfsd/home/.local/bin/anfs-bus`
- anfs-dev: `packages/dev/home/.local/bin/anfs-work` (`new`, `path`, `env`), `spaces/space.yml`
- runtime: `go/internal/{bus,cloudevent,spool}` (02a)

## The format, in `space.yml`

```yaml
id: tkj76y3nfx

access:                          # beyond the job, what this space's dispatched agents may reach
  - {path: ~/.local/share/anfs/sources, mode: read}       # ~ and $VAR expanded; relative: from the space's directory
  - {path: ~/notes/anfs, mode: write}

agents:                          # the roles this space runs; the address is <space path>/<role>
  maintainer:
    harness: claude              # required: claude | opencode | pi | antigravity
    model: opus                  # optional, passed as the harness's model flag
    concurrency: 1               # runs of this role at once (default 1)
    queue: 10                    # runs waiting beyond that are rejected (default 10)
    budget_usd: 5                # claude only (--max-budget-usd); elsewhere a warning at load

dispatch:
  - name: fix-core-report        # required, [a-z0-9-], unique in the space
    on:
      type: report.filed         # required: a glob on the short type; run.* is refused
      subject: "r-*"             # optional glob
      data: {to: file, package: "core/*"}   # optional: a glob per data key, all must match
      space: anfs-dev            # optional: a space spec; default this space (others: open until accepts:)
    as: maintainer               # required: a role of agents:
    harness: claude              # optional: overrides the role's
    job: {name: "fix-{{subject}}", repos: [core]}   # anfs work new <name> <repos>; or none (the default)
    prompt_file: prompts/report-filed.md            # or prompt: | ... (exactly one)
    timeout: 2h                  # default 1h, at most 12h
    rate: 20/h                   # starts per rule (default 20/h)
    stale: 24h                   # an event older than this is not acted on (default 24h)
    retries: 0                   # 0 to 2, after reason exit only (default 0)
```

- **Templates** in `job.name`, `prompt` and the prompt file: `{{id}}`, `{{type}}`, `{{subject}}`,
  `{{time}}`, `{{space}}` (the path), `{{actor}}`, `{{data.<key>}}`, `{{run}}`, `{{job}}`. Plain
  substitution, no logic; an unknown name makes the rule invalid. In `job.name` a value is lowered
  and anything outside `[a-z0-9-]` becomes `-`.
- **Validation**: one parser, in anfsd (`anfsd rules [--json] [space.yml...]`). A rule that fails
  is `invalid` (with the reason) and not loaded; the others load. Refused: a skip-permissions
  harness option, `run.*` in `on.type`, `job:` without anfs-dev/dev installed (`anfs work` is
  missing), a harness not on the service's PATH, `access:` paths that are `/` or `$HOME` itself.
- Where rules come from: every installed space with an id (`wsm ls --json`), read at start and on
  `anfs bus reload` (SIGHUP) or a `source.updated` event.

## A run

1. **Match**: anfsd's durable consumer `anfsd-dispatch` (filters: each rule's `anfs.<space
   id>.<type prefix>`; the globs matched in Go) gets the event. One run per rule and event id: a
   redelivery finds the record and is acked.
2. **Guards**, each a `run.rejected` with its reason: `stale`; `rate`; `queue` (full); `loop` (the
   event's actor is a run of this rule).
3. **Submitted**: the record `~/.local/state/anfsd/runs/<run id>/run.json` and the event's line
   (`event.json`) are written, `run.submitted` emitted, then the message acked.
4. **Started** when the role has a free slot (and the host-wide `ANFSD_MAX_RUNS`, default 2):
   `anfs work new <job> <repos...>` (a job left by an earlier try of this run is reused; any other
   existing job is `run.failed reason=job`), then the harness (below) in its own session, cwd the
   job (or, with `job: none`, the space's directory, read-only), stdin `/dev/null`, output to
   `out.log`, with the environment: `anfs work env <job>`'s variables, `ANFS_ACTOR=<harness>:<last 8
   of the run id>`, `ANFS_AS=<space path>/<role>`, `ANFS_RUN=<run id>`, `ANFS_EVENT=<run dir>/event.json`,
   `ANFS_WORK` unset. `run.working` (data: job, branch).
5. **The prompt** is the rule's, followed by anfsd's footer: who it is (the address), the rule, the
   event (its line), the job's path, what it may not do (below), and "end with a summary of what
   you did and what is left for the person".
6. **Ends**: `run.completed` (exit 0; data: `job`, `commits` per repo ahead of main, `summary`, the
   final answer cut to 1500 bytes); `run.failed` (`reason`: `exit`, `timeout`, `job`, `spawn`,
   `orphaned`; `exit`); `run.canceled` (`anfs agents stop`; `by`). Then `retries`, if any. The job
   stays for the person to review, merge and remove.

All `run.*` events go through the spool into the rule's space; subject the run id; actor `anfsd`.
They are A2A 1.0's task states (submitted, working, completed, failed, canceled, rejected);
`input-required` and `auth-required` arrive with 04.

## Harnesses (`internal/harness`, one file each)

| Harness | Command |
| --- | --- |
| claude | `claude -p --output-format stream-json --verbose --permission-mode dontAsk --settings <run dir>/claude.json --add-dir <each path beyond the cwd> [--model M] [--max-budget-usd N] --name <rule>:<run> -- <prompt>` (corrected at 02b's start against 2.1.296's `--help`: `-p` takes no value and `--add-dir` takes several, so the prompt goes last, after `--`). `claude.json`: `permissions.defaultMode: dontAsk`, `permissions.allow` (`Read` for every path, `Edit`/`Write` for the job, its TMPDIR and the write paths, `Bash`), `permissions.deny` (the list below, as `Bash(<cmd>)` and `Bash(<cmd>:*)`), `sandbox` with `enabled: true`, `failIfUnavailable: true` (a missing backend fails, not an unconfined shell), `allowUnsandboxedCommands: false`, `filesystem.allowWrite` the write paths (keys from 2.1.296's binary). Under `-p` a settings file that fails validation is silently ignored (its `--help`): anfsd writes it from a struct and its tests hold its shape; that 2.1.296 loads it was not run here (no harness is started in this job): the person's first run shows it. The summary is the stream's `result` |
| opencode | `opencode run --format json --dir <cwd> [--model M] <prompt>` with `OPENCODE_CONFIG_CONTENT`: `permission.external_directory` (each access path `allow`, `*` `deny`), `permission.bash` (the denials `deny`, `*` `allow`). The summary is the last text event |
| pi | `pi -p --mode json [--model M] <prompt>`; a rule whose access is only `read` adds `--tools read,grep,find,ls,bash`. Not enforced (plan.md, 5): `anfs agents rules` marks pi rules `unenforced` |
| antigravity | `agy -p <prompt> --sandbox --add-dir <each access path> --print-timeout <timeout>` (macOS). No read/write split: marked `partial`. At 02b's start `agy --help`, `-h` and `--version` (1.3.3) all hung with no output (10 s each, stdin `/dev/null`): the flags are from its binary's strings and changelog, not verified by its help; an expired `--print-timeout` exits 0 with the partial answer, so anfsd's own timeout is what fails the run. `anfs bus status` never asks agy's version |

**What a dispatched agent may not do before 04** (denied to every harness that can deny, and said
in every footer): `git push`; `ppm install|remove`; `anfs install|remove|implode|src update`;
`wsm install|remove`; `anfs report close`; `gh`; `sudo`; `launchctl`; `systemctl`; `anfs bus`;
`anfs agents`; writing outside the job, its TMPDIR and `access:`'s write paths. Merge, install,
push and close are the person's, or this session's acting for them.

The first task of the job: check the claude and agy rows against the installed versions (a
`--settings` file with `sandbox` and `permissions` loads under `-p`; `agy -p` runs headless
without hanging); what does not hold is a finding in the log, and the table is corrected. (Done
with `--help` only, no harness started: the person's instruction for the job. The rows above are
the corrected ones; `02b-log.md` has the findings.)

## Supervision and `anfs agents` (core/anfsd's `anfs-agents`)

As plan.md, 4. Commands:

| Command | Does |
| --- | --- |
| `anfs agents ls [SPEC] [--all]` | `RUN STATUS RULE ROLE SPACE HARNESS JOB AGE`, from the run records (anfsd need not be up); without `--all`, the runs not ended plus the last day's |
| `anfs agents show <run>` | the record, the trigger event, the argv (the prompt cut), the job |
| `anfs agents logs <run> [-f]` | `out.log` (claude's stream rendered as text; `--json` as is) |
| `anfs agents stop <run>\|--all [-y]` | the `cancel` marker, then SIGTERM to the group, SIGKILL after 30 s; anfsd emits `run.canceled`. Asks first without `-y` |
| `anfs agents rules [SPEC]` | `RULE STATUS SPACE ON AS HARNESS`, STATUS `ok`, `invalid` (reason in `--json`), `unenforced`, `partial` |

A run id may be given by its last 8 characters. `anfs bus status` gains `dispatch` (rules loaded,
runs working and queued) and one row per harness a rule uses (found, version).

## The report as a view; ids unique per space

- **The record** moves out of the sink file into STATE: `anfs report new` (file target) writes
  `$ANFS_STATE_HOME/reports/<space id>/<id>.md`, the section as today (`## <id> <title>`,
  `Status: open`, the body), created exclusively (`set -C`): the next free `r-<YYYYMMDD>-<n>` in
  that space's directory, so two sinks or two writers in one space never share an id (the BACKLOG
  collision). The space is the one around the sink file (`--at`, as 01). Then `report.filed`.
- **The sink**: anfsd's consumer `anfsd-sink-report` (`anfs.*.report.>`) runs `anfs report
  __sink` (plumbing) with the event on stdin, one at a time; a failure is redelivered up to 5 times,
  then logged. `__sink` renders into `ANFS_REPORT_TO` when it is a file: on `report.filed` it
  appends the record unless a `## <id> ` heading is already there; on `report.closed` it rewrites
  that section's `Status:` line from the record (today's awk). Under a `mkdir` lock beside the file.
- When anfsd is not running (no live pid in `anfsd.pid`), `anfs report new` and `close` run the
  same render themselves, so BACKLOG.md never waits on a daemon that is not there; the render is
  idempotent, so a daemon that comes up later changes nothing.
- `status` and `close` read and write the record (close: its `Status:` line, then
  `report.closed`); `anfs report show <id>` prints it. GitHub reports are unchanged (the issue is the
  record; filing stays synchronous because it asks the person).
- Reports already in BACKLOG.md have no record: they stay as text, and an open one is closed by
  editing its `Status:` line by hand (no migration code).

## anfs-dev's rules (replacing the grep watcher)

In anfs-dev's `spaces/space.yml`:

```yaml
access:
  - {path: ~/.local/share/anfs/sources, mode: read}
agents:
  maintainer: {harness: claude, concurrency: 1, budget_usd: 10}
dispatch:
  - {name: fix-core-report, on: {type: report.filed, data: {to: file, package: "core/*"}},
     as: maintainer, job: {name: "fix-{{subject}}", repos: [core]},
     prompt_file: prompts/report-filed.md, timeout: 2h}
  - {name: fix-stack-report, on: {type: report.filed, data: {to: file, package: "stack/*"}},
     as: maintainer, job: {name: "fix-{{subject}}", repos: [stack]},
     prompt_file: prompts/report-filed.md, timeout: 2h}
```

A report with no `package` (the owner is core) matches neither; `package: "*"` with `repos:
[core, stack]` is the alternative if the person wants every report taken. `spaces/prompts/report-filed.md`
(new, anfs-dev):

> A bug report, {{subject}}, was filed in anfs-dev: "{{data.title}}" ({{data.package}}). Read it
> with `anfs report show {{subject}}`. Read the AGENTS.md of each repo in your job, and docs'
> ROADMAP "For an agent running this list"; follow it. Reproduce the bug minimally through `anfs
> work run {{job}} ...` with a temp HOME. If you cannot reproduce it, say so and stop: commit
> nothing. Otherwise fix it with a test, run that test file, then the package's suite once, and
> commit on the job's branch with {{subject}} in the message. End with a summary: the cause, the
> commits, the tests run, and what is left for the person (review, merge, install, `anfs report
> close {{subject}} --commit <sha>`).

## Files

- runtime: `go/internal/{space,dispatch,supervise,harness,sink}`, `cmd/anfsd` (`rules`), tests,
  README, AGENTS.md
- core: `packages/anfsd/home/.local/bin/anfs-agents` (new); `anfs-bus` (status rows, `reload`);
  `lib/anfs/report.sh` (the record, `show`, `__sink`, ids); `lib/anfs/events.sh` (`run.*` in the
  completion types); `AGENTS.md` ("Dispatch", "`space.yml`" keys `access:`, `agents:`, `dispatch:`;
  "Reporting a bug"); the skill (filing and `show`); tests
- anfs-dev: `spaces/space.yml`, `spaces/prompts/report-filed.md`
- docs: `02b-log.md`; ROADMAP's "Retiring" line (the grep watcher, BACKLOG as a view, the ids done)

## Test Spec

runtime (Go; fake harnesses and a fake `anfs` as scripts in `testdata/` that record argv, env and
stdin and exit, sleep or spawn a child as told):
- rules: every key valid and invalid; unknown template names; `run.*` refused; a skip-permissions
  option refused; a missing harness; `anfsd rules --json`.
- matching: type glob, subject, data globs, `space:`; the filter subjects of the consumer.
- guards: stale, rate, queue full, loop; one run per rule and event when delivered twice.
- lifecycle: `anfs work new` called with the rendered name and repos, reused on a retry, a foreign
  job fails the run; the harness argv per harness (golden files) and the environment
  (`ANFS_ACTOR`, `ANFS_AS`, `ANFS_RUN`, `ANFS_EVENT`, the job's variables, no `ANFS_WORK`);
  concurrency per role and host; timeout kills the whole group (the fake's child too); a `cancel`
  marker gives `run.canceled`; anfsd restarted adopts a live run and reports a dead one
  `orphaned`; every transition is one valid spool line.
- sink: the event on `__sink`'s stdin, one at a time; a failure redelivered, then logged.

core (bats):
- report: ids from the record directory (two sink files in one space: distinct; ten parallel `new`:
  ten ids); `show`; `__sink` appends once however often it runs; `close` changes the record and the
  file's one line; `status` reads the record; with no anfsd, `new` renders itself.
- `anfs agents`: `ls`, `show`, `logs` from fixture run directories; `stop` on a process group the
  test started (marker written, both processes gone); `rules` through a stub `anfsd`.
- `anfs bus status`: the dispatch and harness rows.

anfs-dev: `anfsd rules` against the job's `spaces/space.yml` gives two `ok` rules.

## Verification

- While working: `anfs work run agents-02b runtime -- sh -c 'cd go && go test -race ./...'`; the
  e2e bats (02a's, plus a rule whose fake harness writes a file: one `report.filed` line in the
  spool gives `run.submitted`, `run.working`, `run.completed` in the stream); `anfs work run
  agents-02b core -- bats packages/anfs/tests/report.bats </dev/null` and anfsd's files
- Once, at the end: core's `packages/anfs/tests/all`; anfs-dev's `bats packages/dev/tests`
- No real harness is started by any test; the claude and agy checks above use `--help`, a
  `--settings` file with `-p "say ok"` in a temp directory, and nothing else (the job's first task)

## By hand, for the person after merge

1. Runtime: merge, push, tag `v0.2.0`; core: the new sums in `packages/anfsd/pins.yml` (this
   session for you).
2. `anfs src update core anfs-dev && ppm install core/anfsd && wsm install anfs-dev` (the prompt
   file's link), then `anfs bus reload`.
3. `anfs agents rules anfs-dev`: two `ok`. `anfs bus status`: `dispatch` and `harness claude` ok
   (the service's PATH finds claude, and claude is logged in for a launchd agent).
4. End the session that greps BACKLOG.md.
5. Your call, it costs one run: file a real small report into anfs-dev and watch `anfs agents ls`,
   `anfs agents logs <run> -f`, then review the job and merge it as before.
6. Any report still open in BACKLOG.md from before: close it by hand when fixed.
