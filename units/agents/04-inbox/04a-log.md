---
plan: agents/04-inbox (04a)
job: agents-04a
date: 2026-10-24
---

# Log: agents/04a, the inbox and approvals (CLI and anfsd)

## What was built

**anfs-io/runtime** (anfsd):

- **The parser** (`internal/space`): `approvals: {merge, install, push}: person|auto` (default
  `person`; another step or value makes it invalid, said, and read as all `person`; the rules
  still load). A rule's `land:` (steps of merge, install, push, merge first, each once, in order;
  needs a job) and `verify:` (an argv, templates per argument, its program on anfsd's PATH;
  required with `land:`, refused without it). `run.auth-required` is one of anfsd's own types (no
  rule on it). `anfsd rules --json` gains `land`, `verify`, `approvals`, `approvals_error`; `anfsd
  match` prints `verify:` (rendered) and `land:`.
- **Verify and the hand-over** (`internal/supervise/land.go`): a run whose rule has `land:` and
  whose harness exits 0 with commits runs `verify:` in the job (`run.working` data `step=verify`;
  the run's allow-listed environment and the job's variables, stdin `/dev/null`, its own session,
  the rule's timeout, output in the run's `verify.log`). Failing: `run.failed reason=verify`
  (detail: the exit and the last line), nothing requested. No commits: completed as without
  `land:`. Then the lander takes the run; the supervisor keeps `LandWorking`, `LandWaiting`,
  `LandNote`, `LandEnd` for it. A record `working` with a `step` or `auth-required` is never
  adopted as a process; a `verify` interrupted by a stop fails the run at the next start.
- **The lander** (`internal/land`, new): approvals in `~/.local/state/anfsd/approvals/<id>/`
  (`approval.json`: the step, who answers, the run, rule, job, trigger and report, per repo
  origin, main, base, head, commits, files, the source's alias, the `space.yml` files a merge
  changes with their top-level keys, the exact `commands`, the report `close`, `by_hand`, `then`
  (what the next steps would run), verify; `commits.txt`, `diffstat.txt`, `verify.log`,
  `step.log`, `answer.json`). Requested (`approval.requested`; the run `auth-required`, no slot
  held), answered (`approval.given` / `.refused`, the answerer's actor), performed with fixed argv
  and no shell, one landing at a time (a process mutex): **merge** (every repo checked first: the
  origin on its main, clean tracked tree, the job branch at the approved head, a fast-forward;
  then `git -C <origin> merge --ff-only <head>`; a later repo failing names what merged, nothing
  rolled back; `source.updated` per source that moved, actor `anfsd`), **install** (`ppm install
  <source>/<name> -y` per installed package whose `packages/<name>/` changed, from ppm's trackers;
  none: skipped, not asked), **push** (main still the approved head; `git -C <origin> push origin
  main:main`). `approval.done` / `.failed`, then the next step, then `run.completed` (`landed`,
  `skipped`). The report closes (`anfs report close <id> --commit <head> --source <alias>`, as
  `ANFS_ACTOR=anfsd`) after install, or after merge when install is skipped or not in `land:`;
  never after push. Refused: the run `canceled` (`reason: refused`, `by` the answerer). Expired,
  never by time: `moved`, `diverged`, `gone`, `canceled` (`anfs agents stop`), `superseded`;
  checked when anfsd would perform, at each rule load (start, SIGHUP, `source.updated`), on each
  SIGUSR1; the run `failed reason=expired` (or `canceled` for a stop). Restart: a given merge
  whose mains equal its heads is done (and its next step requested), anything else `failed`
  (`interrupted`).
- **The policy in force** (`policy/<space id>.json`): first sight all `person`; at each load
  (dispatch's new `Loaded` hook) a differing `approvals:` keeps it and requests a `policy`
  approval once per distinct content; a newer change supersedes it; the file back to what is in
  force supersedes it; a refused content is not asked again. `auto`: anfsd answers itself (actor
  `anfsd`, `via: auto`, the same path), never for a policy, a push under `push: person`, a merge
  that changes a `space.yml` (forced `person (forced: space.yml changed)`), nor a request.
  `--through install|push` gives a run's later steps with its merge (each its own
  `approval.given`, data `through`, `from`).
- **The inbox path** (F6, `cmd/anfsd/inbox.go`): `anfsd inbox answer <id>
  approve|refuse|dismiss [--actor A] [--as SPACEID/ROLE] [--via cli|ui] [--reason T] [--through
  install|push] [-y] [--json]` refuses inside a run (`ANFS_RUN`; a working run's process group or
  session; a process below a working run's pid), checks the item waits, types the id back at
  `/dev/tty` for a policy, a person's push and `--through push` (`-y` and `ANFS_RPC`: 77), writes
  `answer.json` (`dismiss.json` for unhandled work) exclusively, and sends SIGUSR1. The lander
  acts only on that file (via `cli|ui`, never `auto` or actor `anfsd` from a file), checks it
  again, and refuses a bad one by moving it aside (`answer.refused.<n>.json`). An `approval.*` line
  in any spool is a record. `anfsd inbox request merge|install|push --job J [--repo R]...`: a
  request for a job worked in by hand (host scope, the person's, no run, no verify).
- **Unhandled work**: a dismissal is `inbox.dismissed` (actor the answerer; data `run`, `rule`,
  `trigger`, `reason`, `via`), a held run canceled by the answerer; `dismiss.json` becomes
  `dismissed.json`. **A redispatch of an event whose run is held for the rate releases it from
  that hold** (`ReleaseHeld`: marked redispatched, the rate and stale skipped, its queue still
  applies): 02d's own path, so the inbox's "redispatch (now)" of a delayed item has no second one.
- `anfsd run`: SIGUSR1 wakes the lander. `config`: `approvals/`, `policy/`, the sources
  (`$ANFS_DATA_HOME/sources`), ppm's trackers (`$PPM_STATE_HOME/installed`). `harness`: `Denied`
  gains `anfs inbox`; `Secrets` (claude's `denyRead`) gains `$XDG_STATE_HOME/anfs/rpc` beside
  `nats` (02d's, which covers `nats/users`). The footer of a rule with `land:` says anfsd checks and
  asks, and the agent does not merge, install or close. `testfake`: `anfs work new` over origins
  (real worktrees and `.job.yml`), `anfs report close`, a fake `ppm`. README, AGENTS.md.

**core**:

- `lib/anfs/events.sh`: `run.auth-required`, `approval.*`, `inbox.dismissed` in the type list.
  `anfs rpc`: `inbox` is a group (its `ls`, `show` reads; answers and `request` writes).
- **`anfs inbox`** (core/anfsd's `anfs-inbox`, the package seam), from the records with anfsd
  down: `ls [SPEC] [--all]` (`ID STEP SPACE WHAT RUN AGE STATUS`; unhandled work is STEP
  `dispatch`: per rule and event the newest record, rejected for stale, loop, queue or rate, or
  still held; a delayed one that starts leaves by itself), `show <id> [--diff] [--log]` (the step
  and why it is the person's, the trigger and its title, the run and its summary, per repo
  base..head with commits and diffstat, a `space.yml` warning, verify's last 20 lines, the exact
  commands and the close, by hand, what follows, `would expire: moved|diverged|gone`; `--json`
  compact), `approve`, `refuse`, `request`, `redispatch` (through `anfs agents redispatch`;
  `--again` for stale and loop), `dismiss`. Every answer goes through `anfsd inbox answer` with the
  actor resolved by events.sh's rules (`ANFS_ACTOR`, Claude Code, `shell:$USER`; `ANFS_AS`).
  `ANFS_RUN` set refuses approve, refuse, redispatch, dismiss and request.
- `anfs agents`: `ls` shows a landing run's step (`auth-required (merge)`, `working (verify)`);
  `show` its `land:`, `verify:`, approvals, what landed; `stop` on a waiting run writes the marker
  and SIGUSR1 (its approval expires); `redispatch` refused with `ANFS_RUN` set.
- `anfs bus status`: an `inbox` row, last (`0 waiting`, or `N waiting (a approvals, d
  unhandled), oldest 2h; a policy change waits: anfs-dev (anfs inbox)`; never a failure).
- AGENTS.md ("Dispatch": landing, `anfs inbox`; `space.yml`'s keys; "Events"), anfsd's README,
  the `anfs-bug-report` skill (no message after filing; anfsd closes under `land:`). Versions by
  the hook: anfs 0.2.19, core/anfsd 0.3.3.

**anfs-dev**: `spaces/space.yml`: `approvals: {merge: person, install: person, push: person}`;
both rules `land: [merge, install]`; `fix-core-report` `verify: [anfs, work, run, "{{job}}",
core, --, packages/anfs/tests/all]`; `fix-stack-report` the reported package's suite (`sh -c '...'
sh "{{data.package}}"`: the package an argument, never shell text; a package without a suite
says so and passes). The prompt: the agent does not merge, install or close. The comment no
longer says the rules are committed disabled. `space.bats` updated (it still expected the rules
disabled since `e75584b`). dev 0.1.18 by the hook.

**docs**: this log; ROADMAP's 04 line.

## Commits (branch agents-04a, not pushed)

Before committing, runtime was rebased onto main `0cef342` and core and docs fast-forwarded to
main (`06afc0c`, `68ad07a`): every repo's branch is a fast-forward of main.

| Repo | Commit | What |
| --- | --- | --- |
| runtime | `098a0dc` | the parser: `approvals:`, `land:`, `verify:`; `anfsd rules` and `match` show them; the record's landing fields |
| runtime | `b9f77dc` | the lander, verify and the hand-over, the policy in force, the inbox path (`anfsd inbox`), dismissals, the rate-held release, SIGUSR1; testfake; tests |
| runtime | `cb388b7` | harness: `anfs inbox` denied, `anfs/rpc` unreadable; golden files |
| runtime | `0f770b5` | README, AGENTS.md |
| core | `d19dec9` | events.sh types, rpc's `inbox` group, registry's list heading (anfs 0.2.19) |
| core | `1d03773` | `anfs inbox`; `anfs agents` and `anfs bus status`; README; tests (core/anfsd 0.3.3) |
| core | `3a04315` | AGENTS.md, the `anfs-bug-report` skill |
| anfs-dev | `5fb34b3` | `approvals:`, `land:`, `verify:` on both rules; the prompt; space.bats (dev 0.1.18) |
| docs | (this) | this log, ROADMAP |

The runtime commit `cb388b7`'s message also names the footer's landing sentence; that change is in
`b9f77dc` (supervise.go), not in the harness commit.

## Decisions made during the work (deviations from the plan, and why)

1. **`STATUS`, not `STATE`**, as `ls`'s last column: core's registry test refuses a list headed
   `STATE` (lists take `STATUS`). The JSON has both `status` (as shown) and `state` (raw).
2. **How an answer reaches anfsd**: anfsd has no RPC, so `anfsd inbox answer` (the binary as a
   CLI, not the daemon) validates, writes `answer.json` and sends **SIGUSR1**; the daemon reads it,
   validates again and acts. With anfsd down the answer waits for its start.
3. **The terminal check is anfsd's**, not the bash's: `anfsd inbox answer` opens `/dev/tty` and
   wants the id's last 8 typed back; `anfs inbox approve` asks no y/N for such an item and passes
   `-y` on, which anfsd refuses (77). `--through push` needs the terminal too.
4. **`ls` and `show` are bash over the records** (as `anfs agents`), so they need no anfsd; the
   expiry a record would get shows in `show` as `would expire: ...`, and anfsd records it when it
   next looks (an answer, a rule load including `source.updated`, SIGUSR1, its start), never by
   polling.
5. **Not installed by anfsd**: besides containers and anfsd's own source (`go/cmd/anfsd` in a
   repo), the `anfsd` and `nats` packages: installing either restarts the bus under anfsd. They
   are in the record's `by_hand`.
6. **A merge that cannot fast-forward at request time** (main already moved past the job) is
   requested and expires at once (`diverged`): the record and the event say so.
7. **The close is the step's `close`**, separate from `commands`, and its failure does not fail
   the step: the result says `report close failed: ...` (the install did happen).
8. **`anfs inbox request`** is in the host scope (a job has no space); `install` uses the job's
   `.job.yml` base (where it was cut) to find the changed packages; `push` approves main as it is.
   It is refused with `ANFS_RUN` set too (the plan named only the answers).
9. **Redispatch of a delayed item**: 02d refused a redispatch while the event had a run not ended;
   now a run held for the **rate** is released from that hold instead (one held for its queue still
   waits for it). That is the inbox's "redispatch (now)".
10. **Under `auto` no `run.auth-required`** is emitted (nothing waits); `run.working step=<step>`
    while anfsd performs.
11. **The process checks** of `anfsd inbox answer`: the process group and the **session** against
    every working run's group (a child keeps the session even in a new group), and the ancestor
    chain against working runs' pids (a child that called setsid).
12. **Slots**: verify runs in the run's own goroutine, so the run keeps its slot while verifying and
    frees it when it waits (`auth-required`). A run that ended while anfsd was down (or adopted)
    verifies in a goroutine of its own, outside its role's slot.
13. **`run.failed` from landing** carries `reason`, `detail`, `attempt` and the usage, not `exit`
    (the harness's 0 would mislead).
14. **space.bats in anfs-dev was stale** (it expected both rules disabled after `e75584b` enabled
    them); updated with the land/verify checks.
15. **One landing at a time per host** is a mutex in anfsd (only anfsd performs); no per-origin
    file lock: git's own `index.lock` keeps a concurrent `anfs src update` and a merge apart (one
    of them fails, said).

## Tests

All under `anfs work run agents-04a <repo> -- ...` with a fresh temp `HOME` and every `XDG_*` in
the job's TMPDIR (`/tmp/anfs-work-501-1/iso.*`), `ANFS_*`, `NATS_*`, `PPM_*`, `WSM_*`, `CLAUDE*`,
`SSH_AUTH_SOCK`, `GH_TOKEN` unset, PATH the system's plus links to go, gofmt, bats, yq, bash, the
pinned nats-server v2.15.1 and nats, git and perl; `GOPROXY=off`.

- **Go** (`go test -race ./...`; gofmt and vet clean; `GOOS=linux go build` clean): **130 tests**,
  0 failed. `-count=3` on land, supervise, dispatch and cmd/anfsd: none flaky. New: space
  (approvals, land/verify valid and invalid), dispatch (a rate-held run released), land (16: merge
  then install, auto with a skipped install, a `space.yml` forcing person, verify failing and no
  commits, refused, expiry four ways and at perform, a partial merge, an origin not ready, push at
  a terminal only, `--through`, a restart mid-step both ways, an approval line is not an answer
  (a run's forged one stamped with `actorclaimed`, the host's), the policy, a request, dismissals;
  every line valid and under 4096 bytes), cmd/anfsd (`anfsd inbox answer`: `ANFS_RUN`, the run's
  group, usage, unknown, 77 for a policy under `ANFS_RPC` and `-y`, `--through push -y`, refusing a
  policy, the answer written and SIGUSR1 sent, twice refused, not waiting, dismiss; `anfsd inbox
  request`; rules and match show land/verify), and the daemon (a space with `approvals: {merge:
  auto}` gets a policy approval at start; a terminal's answer and SIGUSR1 put it in force; the
  delayed event redispatched and run).
  - **Idle CPU**: `TestRunIdleAndStop` 50 ms over ~11 s (05b: 107 ms); `TestRedispatchAndHeldIdle`
    0 s by `ps` over 10 s with a rate-held run and the lander running.
- **e2e** (`go/test/e2e.bats`, nats-server v2.15.1, `ANFS_CORE_LIB` the job's core): **7 passing**.
- **core**: `packages/anfsd/tests` **44** (inbox.bats 7 new, agents.bats 15 (+2), bus.bats 15
  (+1)); events, rpc, registry, settings pass; `packages/anfs/tests/all` once: **772 passed, 1
  failed**, 56 files (pmm and pcm off). The failure is `pcm-local-env.bats` 10 (cloudflare-tunnel):
  the edge script sources `$HOME/.local/lib/anfs/bash.sh`, which the test leaves to the host's real
  install; in the isolated `HOME` it is missing. It fails the same way on main `06afc0c`
  (checked from `git archive`), so it is not 04a's; see open question 1.
- **anfs-dev**: `packages/dev/tests` **42 passing** (space.bats 3, built from this job's runtime).

No harness, no real ppm, no real merge or install: every landing test merges in temp git
repositories with the fake `ppm` and `anfs`.

## Not run

- Nothing installed, loaded or restarted; the live NATS, anfsd and ui not touched; no `anfs inbox
  approve` against live state; no push.
- The terminal prompt (`/dev/tty`) was not driven by a test (no tty here); its refusals (77) are.
- Linux (built for it; `getsid` through the raw syscall); CI; a release.

## The events spec (anfs-io/events): entries for the person to add to REGISTRY.md

Not edited here (the events repo is the person's). The types follow the grammar, so validators
need no release; the vendored examples are unchanged. In "anfs's catalog (`io.anfs`)":

- **Runs**: `run.auth-required` (`rule`, `job`, `step`, `approval`); `run.working` adds `step`
  (`verify`, `merge`, `install`, `push`) after the harness ended; `run.completed` adds `landed`
  and `skipped` (comma-separated steps); `run.failed`'s `reason` adds `verify`, `merge`,
  `install`, `push`, `expired`; `run.canceled` adds `reason` (`refused`).
- **Approvals (anfsd: the lander)**, a new section. Scope: the run's space (`host` for a request
  with no run). Subject: the approval id (UUIDv7). Actor: `anfsd`, or the answerer for
  `.given`/`.refused` (`actorspace`/`actorrole` when it acts as a role). Data (flat strings; a
  key left out when empty): every type `step` (`merge`, `install`, `push`, `policy`) and `run`,
  `rule`, `job` when from a run;
  `approval.requested` `to` (`person`), `policy`, `trigger`, `report`, and per step `repos`,
  `bases`, `heads`, `commits`, `files` (merge; `<repo>=<value>` comma-separated), `packages`
  (install), `repos`, `remote` (push), `was`, `now` (policy); `approval.given` `via` (`cli`,
  `ui`, `auto`), `through`, `from`; `approval.refused` `via`, `reason`; `approval.expired`
  `reason` (`moved`, `diverged`, `gone`, `canceled`, `superseded`), `detail`; `approval.done`
  `result`; `approval.failed` `reason` (the step, or `interrupted`), `detail`.
- **Inbox**: `inbox.dismissed`, scope the rule's space, subject the run id, actor the answerer,
  data `run`, `rule`, `trigger`, `reason`, `via`.
- **Sources**: `source.updated` may have actor `anfsd` (a merge it performed moved the source).

## By hand, for the person, in this order

No migration code: there are no approvals or policies yet; the first load writes anfs-dev's
policy (all person, which its file says, so no request).

1. **Merge**: fast-forward `main` to `agents-04a` in runtime, core, anfs-dev and docs (anfs-dev's
   touches `spaces/space.yml`: yours, always).
2. **Build anfsd from source** (the pinned v0.4.0 has no lander) and point the service at it:

   ```bash
   cd ~/spaces/anfs-dev/runtime/go && go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
   # ANFSD_BIN=~/.local/share/anfsd/dev/anfsd in ~/.config/anfsd/anfsd.local.env, if not there
   ```

   (Or push runtime, tag `v0.5.0`, and pin it in core's `packages/anfsd/pins.yml`; this session
   can do that.)
3. **Install and restart**: `anfs src update core anfs-dev`, then `ppm install core/anfs
   core/anfsd` (events.sh, `anfs rpc`, `anfs inbox`, `anfs agents`, `anfs bus`), `wsm install
   anfs-dev` (space.yml, the prompt), and `anfs bus restart anfsd` (the new binary: the lander and
   its SIGUSR1). `anfs bus logs anfsd`: `policy of anfs-dev: first seen,
   merge=person,install=person,push=person` and `rules: 2 loaded`.
4. **Check**: `anfs inbox` is empty; `anfs bus status` ends `inbox ok 0 waiting`; `anfs agents
   rules anfs-dev` both `ok`.
5. **A dry check**: `anfs agents match --type report.filed --space anfs-dev --subject
   r-20261024-99 --data to=file --data package=core/pcm --data title="a dry check"` prints
   `verify: anfs work run fix-r-20261024-99 core -- packages/anfs/tests/all` and `land: merge,
   install`. Confirm `bats` and `yq` are on anfsd's service PATH (the verify needs them): `anfs
   agents show` of the first run, or `verify.log`, says if not.
6. **The first real approval** (the rules are on since `e75584b`): when a report is filed,
   `anfs agents ls` shows the run `working`, then `working (verify)` (the core suite, about a
   minute), then `auth-required (merge)`. Then:

   ```bash
   anfs inbox                      # the merge: core +N (r-...)
   anfs inbox show <id>            # commits, diffstat, verify's tail, the exact git merge --ff-only
   anfs inbox show <id> --diff     # the whole diff, if you want it
   anfs inbox approve <id>         # y: anfsd merges (live at once), source.updated
   anfs inbox                      # the install: core/<pkg> (or none: skipped, the report closed)
   anfs inbox approve <id>         # y: ppm install ... -y, then anfs report close
   anfs report status r-...        # available; the run: anfs agents ls (completed)
   ```

   Push by hand when you choose (`anfs src push core`). A session acting for you may answer merge
   and install with `-y` (recorded as its actor); a policy change and a push stay at your terminal.
   If the first verify fails on a test that is not the run's (e.g. open question 1 under anfsd's
   environment), the run ends `failed (verify)`: `anfs agents show <run>` and its `verify.log`.
7. **Withdraw the ping** (plan step 5): delete "Always ping after filing" (and the "messages back"
   sentence) from `~/.claude/projects/-Users-roberto-spaces-cnfs-partitia/memory/anfs-bugs-to-backlog.md`,
   tell the cnfs session, tick the ROADMAP's Retiring line.
8. **REGISTRY.md** in anfs-io/events: the entries above (a minor version of the spec).
9. Only if a rule ever gets `push` in `land:` (or you approve an `anfs inbox request push`): check
   once that anfsd can push under launchd (`SSH_AUTH_SOCK` in its environment).

## Open questions

1. **`pcm-local-env.bats` 10 reads the host's real install** (`$HOME/.local/lib/anfs/bash.sh`
   through the edge script, with the outer HOME): it passes only where core is installed, and
   leaks onto the real HOME. Recommended: a small fix job (the test links `lib/anfs` into its
   HOME, or the edge sources `bash.sh` beside its own library). It also matters now because
   `verify:` runs core's whole suite.
2. **A failed `anfs report close` does not fail the install** (the result says so): recommended as
   is (the install happened; a failed approval would suggest it did not), with `anfs report close`
   by hand. The alternative is a third state.
3. **Approval records are never pruned** (`ls --all` shows 30 days): recommended a later
   `anfs inbox prune` or a size bound with 04b's ui, not now.
4. **The core verify is the whole suite** (about a minute, CPU-heavy, under anfsd): recommended as
   is (merge is the gate that matters); a per-package suite like the stack rule's is the
   alternative if it is too slow.
5. **ANFS_RUN and the process checks are layered, not authentication** (one uid): as the plan
   says, real authentication waits for roles (AGENT-NATIVE's open question 2).
