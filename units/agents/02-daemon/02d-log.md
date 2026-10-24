---
plan: agents/02d-safe
job: agents-02d
date: 2026-10-24
---

# Log: agents/02d, safe to turn on dispatch

## What was built

- **anfs-io/runtime** (anfsd):
  - **A run's environment is an allow-list** (`internal/supervise/env.go`, `runEnv`): a working
    shell, the `XDG_*` paths, `ANFS_*` (not `ANFS_WORK`, `ANFS_WORK_SAID`, `ANFS_EVENTS_SPOOL` or
    the per-run names), and claude's credential for claude only. Git asks nobody
    (`GIT_TERMINAL_PROMPT=0`, an empty `credential.helper` through `GIT_CONFIG_COUNT`/`KEY`/`VALUE`
    after any the job sets). anfsd's own `anfs work new|path|env` calls get the same list.
  - **A run's own spool** (`internal/supervise/ingest.go`): `<run dir>/events/` (0700,
    `ANFS_EVENTS_SPOOL`), drained into the log by `cloudevent.StampLine` (actor, actorspace,
    actorrole, **actorrun**; **actorclaimed** when the line said otherwise; every other attribute's
    bytes kept, `data` last) and `cloudevent.AppendLine`. Refused and counted
    (`events_refused`): not an event, `run.*`, a scope directory that is not the line's source,
    past `ANFSD_RUN_EVENTS_MAX` (1000), over 4096 bytes once stamped. Cursors and counts in
    `<run dir>/cursors.json`. Drained on a `spool.Watch` while the run works, in `end()` before
    the final `run.*` (so `finishDead` too), at adoption, and at start for runs that ended in the
    last day with lines past their cursors; the first batch per spool file after a start skips ids
    already in the last MiB of the target file. `spool.Spool.NoFollow` (a symlinked month file is
    never read). `cloudevent.Parse` reads `actorrun` and `actorclaimed`; the loop guard uses
    `actorrun`. `harness.Spec.Spool`: claude's `sandbox.filesystem.allowWrite` only (not the
    Edit/Write allows), opencode's `external_directory`, antigravity's `--add-dir`. The footer
    names the spool.
  - **What a harness loads** (`internal/harness`): claude `--setting-sources=` (empty) and
    `--strict-mcp-config`, `claudeMdExcludes` (`~/.claude/CLAUDE.md`, `~/.claude/rules/**`),
    `sandbox.filesystem.denyRead` (`harness.Secrets`: `~/.ssh`, `~/.config/gh`,
    `~/.git-credentials`, `~/.netrc`, `~/.claude/.credentials.json`, `$XDG_STATE_HOME/nats`),
    `sandbox.credentials.envVars` (`ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN`: deny),
    `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1`; `ANFSD_CLAUDE_BARE=on` adds `--bare`. pi
    `--no-extensions --no-mcp --no-skills --no-prompt-templates --no-approve`. opencode
    `OPENCODE_DISABLE_PROJECT_CONFIG`, `OPENCODE_DISABLE_CLAUDE_CODE`,
    `OPENCODE_DISABLE_EXTERNAL_SKILLS`, `OPENCODE_PURE`. A rule's `args:` also refuses
    `--mcp-config`, `--strict-mcp-config`, `--plugin-dir(-no-mcp)`, `--bare`, `--safe-mode`,
    `--extension`, `--approve`. Golden files: `claude.golden`, `claude-bare.golden` (new),
    `opencode.golden`, `pi.golden`, `antigravity.golden`.
  - **Backpressure** (`internal/dispatch`, `internal/supervise/hold.go`): `overflow: delay|reject`
    and `hold:` per rule; the guards in order (invalid job, stale, loop, then rate and queue);
    held runs (`submitted` + `held`, `held_since`; `run.submitted` data `held` and `detail`)
    released oldest first on each `schedule()` and by one timer per rule while the rate holds
    them; `rate:` counts `queued`; stale checked again at start (`run.rejected reason=stale`,
    redispatched runs exempt); `Has` split into `Seen` and `Handled` (+ `Unended`); held runs
    survive a restart; redispatch requests (`~/.local/state/anfsd/redispatch/<uuid>.json`) read at
    start and after each reload. `anfsd rules --json` gains `overflow` and `hold`; `anfsd match`
    says `hold`.
  - **`dispatcher: anfsd|none`** (`internal/space`): rules `off`, not loaded nor in the filters,
    queued and held runs canceled `by=dispatcher:none` at each reload, redispatch refused.
  - `internal/config`: `ANFSD_RUN_EVENTS_MAX`, `ANFSD_CLAUDE_BARE`. `anfsd match` prints
    `ANFS_EVENTS_SPOOL`, the git variables and the names of anfsd's environment that pass.
    `testfake`: the fake harness and anfs find their directories beside `HOME` (FAKE_* do not
    pass the list), the fake harness records the environment it got and has a `script` mode.
    README, AGENTS.md.
- **core**:
  - `lib/anfs/events.sh`: `ANFS_EVENTS_SPOOL` (emits go to the run's spool, the 512-byte reserve).
  - core/anfsd's `anfs agents`: `ls` shows `held:rate`/`held:queue`; `ls --unhandled`; `show`
    (held since, redispatched by, events ingested and refused, bytes not ingested); **`redispatch
    <event> [rule] [--again] [-y]`**; `rules` shows `off`. `anfs bus status`'s dispatch row: the
    `off` rules and the held runs. `anfsd.env`: `ANFSD_RUN_EVENTS_MAX`, `ANFSD_CLAUDE_BARE`.
  - AGENTS.md ("Dispatch": `dispatcher:`, `overflow:`, `hold:`, the environment, the enforcement
    table, what a harness loads, backpressure, redispatch, the harness table; "Events": a run's
    spool, the stamp, `actorrun`, `actorclaimed`; `space.yml`'s keys; `anfs bus status`), anfsd's
    README.
- **docs**: this log.

## Commits (branch agents-02d, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| core | `4a0d90e` | events.sh: `ANFS_EVENTS_SPOOL` and the reserve, bats (anfs 0.2.18, by the hook) |
| runtime | `b2e085a` | the env allow-list, git's helper reset, tests |
| runtime | `c0833e8` | the per-run spool, ingestion and the stamp, the drains, `Spec.Spool`, golden files, the e2e |
| runtime | `ff29164` | settings isolation: claude's flags and `claude.json` keys, `ANFSD_CLAUDE_BARE`, pi, opencode, golden files |
| runtime | `b1645cf` | backpressure: `overflow:`, `hold:`, held runs, `Seen`/`Handled`, stale at start, redispatch requests, the restart |
| runtime | `c545fe7` | `dispatcher: none` |
| core | `01ac16e` | `anfs agents redispatch`, `ls --unhandled`, the statuses, `anfs bus status`, anfsd.env, AGENTS.md, tests (anfsd 0.3.1, by the hook) |
| docs | (this) | this log |

## The job's first task (no harness started: docs, `--help` text inside binaries, strings)

Claude Code was **not run**, not even `--help` (the person's rule for this job). Checked against
code.claude.com's cli-reference, settings, settings-reference, sandboxing, memory and env-vars pages
(fetched 2026-10-24), and the installed 2.1.296 binary's strings:

- **`--setting-sources ''` works.** In 2.1.296 the flag's parser splits the value on commas and
  returns an empty list for `""` (no throw); `--restricted` itself calls it with `""`; and Claude
  Code forwards the flag to the sessions it starts as `--setting-sources=<list>` (the `=` form,
  empty when the list is). So anfsd passes **`--setting-sources=`**: no user, project or local file,
  only `--settings` and managed policy (`flagSettings` and `policySettings` are always added). The
  person's decision says `--setting-sources local`; the plan said to use the empty list if accepted,
  which it is (deviation 1).
- `--strict-mcp-config`: "Only use MCP servers from `--mcp-config`, ignoring all other MCP
  configurations"; anfsd passes no `--mcp-config`, so none connect.
- `--bare` (and `CLAUDE_CODE_SIMPLE`): "OAuth tokens and keychain credentials are not read, so
  Anthropic authentication must come from `ANTHROPIC_API_KEY` or an `apiKeyHelper`". As the plan
  said: opt-in.
- `claudeMdExcludes`: "Patterns are matched against absolute file paths using glob syntax", any
  settings layer, arrays merge; it cannot exclude a managed CLAUDE.md.
- Sandbox paths are plain absolute paths (`/tmp/build`, `~/.kube`), unlike permission rules'
  `//path`. `sandbox.credentials.envVars` is a list of `{"name": ..., "mode": "deny"|"mask"}`;
  `deny` unsets the variable before each sandboxed command. `sandbox.filesystem.denyRead` exists.
  Under `dontAsk`, a host outside `allowedDomains` (empty) is refused (the sandboxing page's table
  is for interactive sessions; `-p` is not mentioned: assumed the same, the first run shows it).
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` strips credentials from Bash, hooks and stdio MCP servers by
  name or value, **but leaves `GITHUB_TOKEN`/`GH_TOKEN` and `GIT_CONFIG_*` in place**: fine here,
  since the allow-list never passes the GitHub tokens, and the git reset must survive.
- Also found: `--safe-mode` (2.1.296) turns off CLAUDE.md, skills, plugins, hooks, MCP, agents and
  auto memory but keeps authentication. Not used: it also drops the repos' CLAUDE.md, which the plan
  keeps. A choice for later if user skills and agents must go too without an API key.
- The keys `claudeMdExcludes`, `denyRead`, `envVars`, `SUBPROCESS_ENV_SCRUB` are all in 2.1.296's
  strings.
- **opencode 1.18.35** (binary strings): `OPENCODE_CONFIG_DIR` **adds** a directory (its
  `opencode.json` and `.opencode` content) beside the global `~/.config/opencode`, which is always
  read: it does not replace it, so it cannot isolate. `OPENCODE_DISABLE_PROJECT_CONFIG` skips only
  the project's `opencode.json[c]` files; `OPENCODE_CONFIG_CONTENT` is still merged last.
  `OPENCODE_PURE=1` ("skip external plugins entirely") was found and added. The global config (its
  MCP servers) has no switch: documented, `partial` as before.
- **pi 1.1.0**: the five flags are in its `docs/cli.md` (`-ne`, `--no-mcp`, `-ns`, `-np`, `-na`).
  Not run.
- **`anfs work new` needs no network**: `anfs-work` makes worktrees of local clones and reads `wsm
  repo list --json`; no fetch. So its calls get the allow-list too.

## Decisions made during the work (deviations from the plan, and why)

1. **`--setting-sources=`** (the empty list), not `local`: see the first task. The `=` form is the
   one Claude Code forwards itself.
2. **`claudeMdExcludes` also excludes `~/.claude/rules/**`**: user rules are the same kind of
   personal instruction as `~/.claude/CLAUDE.md`.
3. **`OPENCODE_PURE=1`** added to opencode's environment (found in the first task).
4. **`cloudevent.StampLine`**, not `Stamp`: `Stamp` is the struct. The stamped line is rebuilt in
   the profile's order (`actorrun` and `actorclaimed` after `actorrole`), attributes the profile
   does not know after `anfsv`, `data` last; a line with a doubled key is refused.
5. **The claim's text**: `<actor>` plus ` as <actorspace>/<actorrole>` when the line named a role;
   a role without a space reads ` as /person`. A line with no `actorspace`/`actorrole` of its own
   is not a claim about them (anfs_emit leaves them out when `ANFS_AS` is unset or unresolved).
6. **The counts** (`#ingested`, `#refused`) live in the run's `cursors.json` beside the cursors, so
   they are saved atomically with them and survive a crash; `run.json` gets them at the run's end
   (`events_ingested`, `events_refused`). `anfs agents show` reads the record, so a working run's
   counts show at its end. An invalid line counts as refused.
7. **A failed append stops the batch** without moving the cursor (tried again at the next drain);
   the plan only described refusals, which do move it.
8. **Redispatch request format**: `{event, rule, by, again}` (the plan named the first three;
   `--again` has to reach anfsd). Requests are written to a dot-file and renamed. A rule with a
   run of the event not ended (queued, held, working) is always refused, `--again` or not: two
   runs of one event at once is never wanted. A redispatched run is held only for its role's
   queue (never behind the rule's rate-held runs), and in hold it may go by rate-held runs.
9. **Held runs leave hold on one condition**: their role's queue has room **and** their rule's
   rate has a start free (a redispatched run skips the rate). The `held` label stays what it was
   at submission. Once a rule's oldest run is held by the rate, its later ones wait behind it.
10. **The rate timer** is `time.AfterFunc` to the oldest counted start + window (+10 ms); the
    tests use a 1 s window and the real clock instead of a fake one.
11. **`dispatcher:` with another value** (`firstmate`) makes the space's rules `invalid`, saying a
    named tool waits for the `tools:` slots. The plan said "only anfsd and none are accepted"; it
    also said another tool "simply means none", which a typo should not silently become.
12. **`dispatcher: none` cancels by the cancel marker** (`by` `dispatcher:none`), so the scheduler
    ends those runs as it ends any canceled one; `anfsd match` (a supervisor only opened) writes
    no marker.
13. **`anfs bus status`** says `N off (dispatcher: none)` only when there are some; `held` is
    always said.
14. **`anfsd match`** shows `passes:` (the names, never values, of anfsd's environment that reach
    the run) and the git variables, and its result `hold` for a run that would be held.
15. **The e2e's dispatch tests keep the fakes' records beside `HOME`** (`log/`, `ctl/`, `work/`),
    since `FAKE_*` do not pass the allow-list either; `cmd/anfsd`'s tests link them there.
16. The plan's "Crash between append and cursor ... and the shipper on embedded JetStream publishes
    each once": the Go test proves each id once in the month file (the tail check; a mutation that
    disables it fails the test). The shipper's once-per-id on the bus is 02a's test (`Nats-Msg-Id`
    and the 24 h window), not repeated here.
17. The docs worktree was missing from the job; it was added with `anfs work add agents-02d docs`
    (the job's tool), only for this log.

## Tests

- **Go** (`go test -race ./...` in `runtime/go`, then three runs with `-count=3`; gofmt and vet
  clean): **97 tests** (02c: 76). Three runs with `-count=3` all passed: none flaky.
  - New: the env allow-list (claude and opencode; a `HOME` gitconfig whose helper hands out a
    password gives none, `git credential fill` exits 128; the job's `GIT_CONFIG_COUNT` kept; anfs's
    calls), `StampLine` byte for byte (honest, forged, unknown attributes, a doubled key, over 4096
    once stamped), `AppendLine`, a run's three lines (honest, forged, `run.completed` refused)
    stamped and before its `run.completed`, refusals (scope, symlink, the maximum, size, not an
    event) and a partial line, the crash repeat (each id once), `finishDead`, adoption mid-run,
    the loop through `actorrun`, harness golden files and `claude.json`'s keys, config, `args:`
    refusals, `overflow: reject`, held for the queue (in order), `hold: 2 full`, held for the rate
    (released after the window, a timer only while holding), stale at start and the redispatch
    exemption, held runs across a restart (in order), a held run canceled, a NATS redelivery of a
    rejection (seen, not handled), redispatch (taken, refused, `--again`, stale skipped, disabled,
    no rule), `dispatcher: none` (space, dispatch, supervise, `rules` and `match` say `off`).
  - **Idle CPU**: `TestRunIdleAndStop` 102 ms of CPU over ~11 s, start included (02c: 82 ms) (limit 200 ms in 10 s); new
    `TestRedispatchAndHeldIdle`: with one run held on a rate timer, 10 ms over 10 s by `ps`; a
    request at start and a request with SIGHUP.
- **e2e** (`go/test/e2e.bats`, the pinned nats-server v2.15.1 and nats v0.5.0 linked into the job's
  TMPDIR, `ANFS_CORE_LIB` the job's core `lib/anfs`): **6 passing** (+1). The fake harness emits
  `report.closed` through core's real `anfs_emit` (also passing with a `printf`'d line when
  `ANFS_CORE_LIB` is unset): on the bus with `actor` `claude:<8>`, `actorspace`, `actorrole`
  `maintainer`, `actorrun` ending in those 8, no `actorclaimed`; the forged run's
  (`ANFS_ACTOR=shell:person ANFS_AS=anfs-dev/person`) arrives with the stamp and `actorclaimed`
  `shell:person as abcde23456/person`; the third run's environment has no `SSH_AUTH_SOCK` or
  `GH_TOKEN` though anfsd had them, and has `ANFS_EVENTS_SPOOL` and `GIT_TERMINAL_PROMPT=0`.
- **core**: `events.bats` 22 (+2), `packages/anfsd/tests` 33 (agents 13, +5; bus and anfsd
  updated), and `packages/anfs/tests/all` once at the end: **747 passed, 0 failed**, 54 files (pmm and pcm off), 57 s (02c: 741). The first full run found two suites the plan did not name: `settings.bats` (`ANFS_EVENTS_SPOOL` is a per-process variable like `ANFS_ACTOR`, listed with them) and `registry.bats` (`ls --unhandled`'s heading is one of the lists that are not of resources); both fixed in the commits they belong to.

## Not run

- No harness was started (claude, opencode, pi, agy), not even `--help`; no real podman.
- Nothing was installed, loaded or restarted; the running bus and anfsd were not touched.
- anfs-dev's `bats packages/dev/tests` (anfs-dev is not in this job; its rules are unchanged and
  the new keys have defaults).
- pmm's and pcm's suites (off); CI and the release (nothing pushed); Linux (the `ps -o time=`
  parse in the new cmd test handles procps's format, untested there).

## By hand, for the person after merge

No migration code. Records from 02b keep their meaning: their `queue` and `rate` rejections are
unhandled events now (`anfs agents ls --unhandled`), which `redispatch` can take; their `rate`
counts used `submitted` (read as `queued` when a record has none). A run working across the
upgrade has no spool and the old environment, and ends as before.

1. **Rebuild anfsd and restart it** (the release and core's pins can follow; `pins.yml` still
   names v0.3.0, never tagged: tag runtime's main `v0.3.0` after this merge, or `v0.4.0` and bump
   the pin; this session can do that):

   ```bash
   cd ~/spaces/anfs-dev/runtime/go         # main, with agents-02d merged
   go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
   anfs bus restart anfsd
   anfs bus logs anfsd                      # "rules: 0 loaded, 2 disabled, 0 off (dispatcher: none), 0 invalid", consumers ready
   ```

2. **Core**: `anfs src update core && ppm install core/anfsd` (anfs-agents, anfs-bus, anfsd.env),
   then `anfs bus status`: `dispatch ... 0 working, 0 queued, 0 held`.
3. **A dry check**:

   ```bash
   anfs agents match --type report.filed --space anfs-dev --subject r-20261024-99 \
     --data to=file --data package=core/pcm --data title="a dry check"
   ```

   It shows `--setting-sources= --strict-mcp-config` in the command, `ANFS_EVENTS_SPOOL=<run
   dir>/events` and the git variables in `env`, the `passes:` names (no `SSH_AUTH_SOCK`,
   `GH_TOKEN`), and `disabled: would submit`.
4. **Turn on anfs-dev's rules** (your decision; each report then costs a run, at most 10 USD): as
   02b's log step 5 (delete `enabled: false,` from both rules, commit, `anfs src update
   anfs-dev`). `anfs agents rules anfs-dev` then shows both `ok`. End the session that greps
   BACKLOG.md.
5. **The first real run is the live check** of what no test could do:
   - It starts at all: `anfs agents logs <run>`'s first line `[session ..., <model>]`. If claude
     rejected `--setting-sources=` it fails at once with "Invalid --setting-sources flag" in
     `out.log`: then change it to `--setting-sources local` (one line in `harness/claude.go` and the
     golden file) and say so.
   - **What loaded**: claude under launchd uses the subscription login, and `claude.json` loaded:
     a refused tool in `anfs agents logs <run>` means the permissions are in force.
     To see the setting sources yourself: in the run's job directory start `claude --settings
     <run dir>/claude.json --setting-sources= --strict-mcp-config` interactively and type
     `/status`: "Setting sources" lists only the command line (and a managed source, if any),
     never User or Project settings; `/mcp` shows no server; `/memory` does not list
     `~/.claude/CLAUDE.md`.
   - No herdr hook ran: herdr shows no pane for the run.
   - Its emits arrive stamped: `anfs events ls --type 'report.*' --json | tail -n 3` shows
     `actorrun` on any event the run wrote; `anfs agents show <run>` says `its events: N ingested,
     0 refused`.
   - **The environment**, at the cost of one small run: in a scratch space with an id, a rule with
     `job: none`, `budget_usd: 0.5` and the prompt "Run `env | sort` and `git config --get-all
     credential.helper`, and print both". Its summary must show no `SSH_AUTH_SOCK`, `GH_TOKEN` or
     `GITHUB_TOKEN`, `ANFS_EVENTS_SPOOL` set, and the helpers ending with an empty one (the reset).
     Remove the rule afterwards.
6. **Strict isolation with an API key** (optional): `ANTHROPIC_API_KEY` in the service's
   environment (or an `apiKeyHelper`), `ANFSD_CLAUDE_BARE=on` in `~/.config/anfsd/anfsd.local.env`,
   then `anfs bus restart anfsd`. Billed to the API.
7. **Unhandled events from before**: `anfs agents ls --unhandled`; take one with `anfs agents
   redispatch <event id> [rule]` (it shows the dry check and asks).

## Open questions

1. `anfs work env` unsets `ANFS_WORK`; should it also unset `ANFS_EVENTS_SPOOL`, so a run's
   `anfs work run <job> -- <tests>` does not put the tests' own emits into the run's spool (stamped
   as the run, counted against its 1000)? Core's test helpers set `ANFS_EVENTS=off`, so today it is
   only a test that turns events on without `env -i`. It is anfs-dev's `anfs-work`, not in this job.
2. The sandboxing docs give `dontAsk`'s "refused" for hosts outside `allowedDomains` for interactive
   sessions; `-p` is not mentioned. If a first run shows a network call going out, add
   `sandbox.network.allowedDomains: []` explicitly or a deny.
3. opencode's global config (its MCP servers, `.opencode` directories) and antigravity have no
   switch: both stay documented, not isolated. A `CLAUDE_CONFIG_DIR`-style separate home for them
   would be the next step if they are ever used for dispatch.
