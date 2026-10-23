# Plan 04 — the inbox: approvals as events, `anfs inbox`, per-space `approvals:`

Written 2026-10-23 against core `44e456b` (02a, 02b, 02c, 03 merged) and runtime `ae71386`, in
the job `agents-04-plan` (docs only; code read, nothing run but `anfs bus status`). Binding:
`AGENT-NATIVE.md` ("Decided" above all: approvals are per-space policy `approvals: {merge,
install, push}: person|auto`; no agent changes its own space's approvals; actor and role are
separate fields in events; the full role model is later; answered by the CLI first, then the ui,
then OS notifications), `units/agents.md`, plan 02 and 02b (and its log: what a dispatched agent
may not do before 04), 03 and its log, `reviews/claude-settings.md`. Nothing below reopens them.

**For the person's review.** Each section is a decision: the recommendation first, the
alternatives in a line. The open questions are at the end.

## Context — read these files first

The job would be `anfs work new agents-04 runtime core anfs-dev docs`.

- docs: `AGENT-NATIVE.md`, `units/agents/02-daemon/{plan,02b-dispatch,02b-log}.md`,
  `units/agents/03-ui/{plan,log}.md`, `reviews/claude-settings.md`, this plan
- runtime: `go/internal/{space,dispatch,supervise,harness,cloudevent}` (02b: rules, runs, the
  `Denied` list, claude's settings file; `cloudevent`: data is `map[string]string`, a line is at
  most 4096 bytes)
- core: `AGENTS.md` ("Dispatch", "Events", "The bus", "The toolkit API", `space.yml`'s keys),
  `packages/anfsd/home/.local/bin/anfs-agents` (the shape `anfs-inbox` copies),
  `lib/anfs/events.sh` (`_anfs_ev_actor`), `lib/anfs/report.sh` (`close`), `containers/ui`
  (`lib/anfs/poller.rb`, `app.rb`'s `r.post("action")` and its origin check,
  `ANFS_UI_REMOTE_ACTIONS`)
- anfs-dev: `packages/dev/home/.local/bin/anfs-work` (a job's worktrees are of the live clones
  under `~/.local/share/anfs/sources/` and of the clones a space makes), `spaces/space.yml`

## Overview

After 02b a dispatched maintainer fixes a report in a job, commits, and stops; landing it (merge,
install, push, close) is done by the person, or by a session acting for them, by hand. 04 makes
landing a declared, recorded, event-driven step:

1. A dispatch rule says which steps follow a run (`land: [merge, install, push]`), and optionally
   a check anfsd runs first (`verify:`).
2. When the run ends with commits, anfsd records an **approval** for the first step and emits
   `approval.requested`. The space's `approvals:` says who answers: `person` (it waits in `anfs
   inbox`) or `auto` (anfsd answers itself, on the record).
3. Given, **anfsd performs the step** with fixed commands it listed in the request, emits the
   result, and requests the next step. The run is `auth-required` while it waits (A2A).
4. The space's `approvals:` takes effect only when the person has accepted it: anfsd keeps the
   last accepted policy per space in STATE, and a change in the file is itself an approval that
   only a person at a terminal can give.
5. anfs-ui shows the inbox and can answer merge, install and push through the write path it
   already has (04b).

Done when: a report filed into anfs-dev, with the rules on, ends as a pending merge in `anfs inbox`
showing its commits, diffstat, checks and the run's summary; `anfs inbox approve` lands it; the
report closes; cnfs's `anfs report wait` returns; nobody sent a message.

## 1. The events and what is approved

**Recommendation.** One noun, `approval`, for every step; the step is data. Types:

| Type | When | Actor |
| --- | --- | --- |
| `approval.requested` | anfsd recorded a request (after a run, or `anfs inbox request`) | `anfsd` |
| `approval.given` | answered yes: by the person (`anfs inbox approve`, the ui) or by policy `auto` | the answerer (`shell:roberto`, `claude:5a52...`, or `anfsd` for auto) |
| `approval.refused` | answered no | the answerer |
| `approval.expired` | what was asked no longer holds (below); never answered | `anfsd` |
| `approval.done` | anfsd performed the step | `anfsd` |
| `approval.failed` | performing it failed | `anfsd` |

An approval's states: `requested` → `given` → `done` | `failed`; `requested` → `refused` |
`expired`. All in the space the run is in; subject the approval id (UUIDv7; named by its last 8
characters, like runs). `actorrole` is set when the answerer acts as a role (`ANFS_AS`), else
absent: the person has no role until roles exist.

**What is approved.** Steps: `merge`, `install`, `push`, and `policy` (section 4). The event
carries the ids and the exact objects; everything long lives in the record. Data is flat strings
(01's profile, 4096 bytes a line):

| Key | Example | In |
| --- | --- | --- |
| `step` | `merge` | all |
| `to` | `person` (later: a role address, `anfs-dev/reviewer`) | requested |
| `policy` | `person`, `auto`, or `person (forced: space.yml changed)` | requested |
| `run`, `rule`, `job` | the run id, `fix-core-report`, `fix-r-20261023-4` | all, when from a run |
| `trigger`, `report` | the trigger event's id; its subject when it is a `report.*` | requested |
| `repos` | `core,stack` | merge, push |
| `bases`, `heads` | `core=870eece…` (full shas): main at request time, the job branch's tip | merge |
| `commits`, `files` | `core=3`, `core=7` | merge |
| `packages` | `core/pcm,core/anfs` (installed packages whose `packages/<name>/` the merge changed) | install |
| `remote` | `core=origin main` | push |
| `was`, `now` | `merge=person,install=person,push=person` | policy |
| `via` | `cli`, `ui`, `auto` | given, refused |
| `reason` | free text (cut to 500 bytes), or `moved`, `diverged`, `gone`, `canceled`, `superseded` | refused, expired, failed |
| `result` | `core=870eece..4b1c2d0` | done |

**The record**: `~/.local/state/anfsd/approvals/<id>/` — `approval.json` (the above, state,
times, the exact commands approve will run), `commits.txt` (`git log --oneline base..head` per
repo), `diffstat.txt`, `verify.log`, `step.log` (what performing printed), `answer.json`. Like
run records: `anfs inbox` reads them with anfsd down.

**Tied to the run** by `data.run` (and the run's record gets `approvals: [ids]`), and to what
started the run by `data.trigger` and `data.report`. One request per step, in order merge →
install → push; the next is requested when the previous is `done`.

**Expired**, never by time in 04: `moved` (the job branch's tip is not the approved head),
`diverged` (main moved and a fast-forward is no longer possible), `gone` (the job was removed),
`canceled` (`anfs agents stop` on the run), `superseded` (a newer policy request for the same
space). Each is detected when anfsd would perform or when the record is read, never by polling.

*Alternatives:* a noun per step (`merge.requested` → `merge.approved`, AGENT-NATIVE's first
sketch: six types times three steps, and a fourth step means new types); no `.done`/`.failed`
(the result only in `run.*`: a request from `anfs inbox request` has no run); a time expiry
(nothing goes stale by itself while the shas hold).

## 2. Who acts when an approval is given

**Recommendation: anfsd performs the step**, under the space's policy, with fixed argv and no
shell, and only the commands written into the request before it was answered:

| Step | anfsd runs (in each origin, under a per-origin lock, one landing at a time per host) |
| --- | --- |
| merge | checks first, for every repo: the origin is on its main branch with a clean tracked tree, the job branch's tip is the approved head, main is an ancestor of it; then `git -C <origin> merge --ff-only <head sha>` per repo. Then `source.updated` for each source that moved (dispatch reloads its rules, the ui its sources), and, when the trigger is a report, `anfs report close <id> --commit <head> --source <repo>` |
| install | `ppm install <source>/<name> -y` for each installed package the merge changed (never `-r`). Changed containers and anfsd's own source are listed in the record as "by hand" and not run. No such package: the step is skipped, recorded, not asked |
| push | `git -C <origin> push origin <main>:<main>` per merged repo with a remote; never `--force`, no tags |

Note what merge means here: a job's worktrees are of the **live** clones under
`~/.local/share/anfs/sources/`, and tools are linked from them, so a fast-forward there is live at
once. `merge` is the gate that matters; `install` adds the post-install hooks.

A partial failure (the second repo's merge fails after the first's) is `approval.failed` naming
what was merged; nothing is rolled back (no migration code, no rewriting history). anfsd restarted
mid-step: a merge whose main already equals the head is `done`; anything else `failed` with
`reason: interrupted`.

**Why:** between the person's yes and the action there is no model, only the commands they were
shown; the record and `approval.done` say exactly what ran with which shas; a dispatched agent
keeps every 02b denial (it never gets merge, install or push); and `auto` is the same code path
with a different answerer.

*Alternative:* a dispatched agent performs the step after seeing `approval.given` (a rule on
`approval.given`): the denials would have to be lifted per run, the agent could do more than was
approved, and the audit is the agent's own account. Kept out.

## 3. `anfs inbox`

core/anfsd's `anfs-inbox` (beside `anfs-agents`), `--json` on every command:

| Command | Does |
| --- | --- |
| `anfs inbox [ls] [SPEC] [--all]` | `ID STEP SPACE WHAT RUN AGE STATE`; pending items addressed to the person, `--all` the last 30 days. WHAT: `core +3 (r-20261023-4)`, `core/pcm, core/anfs`, `origin main` |
| `anfs inbox show <id> [--diff] [--log]` | the step and why it is the person's; the trigger (report id and title); the run (rule, role, harness, status, its summary); per repo base..head, the commits, the diffstat, a warning when a `space.yml` changed; `verify:`'s result (exit, the last 20 lines, the log's path); **the exact commands approve will run**; then what the next steps would be. `--diff` the full diff, `--log` the run's log (`anfs agents logs`) |
| `anfs inbox approve <id>... [--through install\|push] [-y]` | asks unless `-y`; `--through` gives the later steps of the same run now (each recorded as its own `approval.given` when requested) |
| `anfs inbox refuse <id> [--reason TEXT] [-y]` | the run ends `canceled`; the job stays |
| `anfs inbox request merge\|install\|push --job J [--repos R...]` | a request without a run, for a session that worked in a job by hand (this orchestration's ff-merge, through the inbox). Requesting is not approving |

- **Addressing**: every item is `to: person` in 04, and `ls` lists those. When roles come, `to`
  becomes a role address and `ls` takes `--as`; nothing else changes.
- **`verify:`** (a rule key, optional): an argv anfsd runs in the job after the run exits 0 and
  before the merge request, e.g. `[anfs, work, run, "{{job}}", core, --, packages/anfs/tests/all]`.
  A failure ends the run `failed` (`reason: verify`) with no request. Without it, show says
  "no check run by anfsd; the run's summary says: …".
- `anfs bus status` gains an `inbox` row (`3 waiting, oldest 2h`; a waiting policy change named).
- A row of the `anfs agents ls` table shows `auth-required` for a run waiting on the person.

*Alternatives:* `show` without the commands to run (the person approves a description, not an
action); anfsd always running the package's suite (slow, and the suite differs per repo: `verify:`
lets the rule say); no `request` (sessions keep merging by hand, outside the record).

## 4. `approvals:` in `space.yml`, and nobody changes their own

```yaml
approvals:            # who answers each step: person (the default) or auto
  merge: person
  install: person
  push: person
dispatch:
  - name: fix-core-report
    ...
    land: [merge, install]          # the steps that follow a run with commits (default: none, as 02b)
    verify: [anfs, work, run, "{{job}}", core, --, packages/anfs/tests/all]
```

Parsed by 02b's one parser (`internal/space`): unknown steps or values make the space's
`approvals:` invalid (shown by `anfs agents rules`, and treated as all `person`).

**Enforcement: the effective policy is not the file, it is the last one the person accepted.**

- anfsd keeps `~/.local/state/anfsd/policy/<space id>.json` (the accepted values, by whom, when,
  the file's sha). A space first seen gets all `person`.
- At every rule load (start, `anfs bus reload`, `source.updated`), anfsd compares the file's
  `approvals:` with the accepted one. Equal: nothing. Different: the accepted one stays in force,
  and anfsd requests a `policy` approval (`was`, `now`), once per distinct content; a newer change
  supersedes it.
- A `policy` approval is always the person's, whatever the policy says, and is answered only at a
  terminal: `approve` reads the confirmation from `/dev/tty` (the id typed back); `-y` is refused,
  the toolkit API answers it 77 (needs a terminal), the ui shows the command.
- So it does not matter how the file changed: a job's commit merged, a direct edit in the source,
  a new space. A space.yml lives in a source repo an agent can edit in a job; editing it changes
  nothing until the person answers at a terminal.
- And belt and braces: a **merge whose diff touches any `space.yml` is forced to `person`**, even
  under `merge: auto` (`policy: person (forced: space.yml changed)`), since `access:`, `agents:`
  and `dispatch:` widen what agents may do too, and show names the keys that changed.

**`auto`** means anfsd answers: `approval.requested` then `approval.given` (actor `anfsd`, `via:
auto`) then the step, so the record is the same shape. Two limits: `merge: auto` on a rule with no
`verify:` acts as `person` (`anfs agents rules` says so), and `auto` never applies to `policy` or
to a forced merge.

**Who may answer** (one host, one uid: layered, not absolute; real authentication is roles' and
AGENT-NATIVE's open question 2):
1. Dispatched runs are denied `anfs inbox approve|refuse` (added to `Denied`), and claude's
   sandbox gets `denyRead` on the secrets an answer could go around with
   (`~/.local/state/anfs/rpc`, `~/.local/state/nats/users`) — the keys `denyRead` and
   `allowUnixSockets` are in 2.1.296's binary, but their behaviour is to check at 04's start, as
   02b did.
2. `anfs inbox approve|refuse` refuse when `ANFS_RUN` is set; the answer is a file in the record
   (`answer.json`, made exclusively) that a sandboxed run cannot write; anfsd validates it and
   emits `approval.given`/`.refused` with the answerer's actor.
3. `policy` needs a terminal (above).
A session acting for the person (this one) may answer merge, install and push with `-y`; the
event records its actor (`claude:…`), so the person sees who did.

*Alternatives:* the policy outside the repo (`~/.config/anfs/approvals/<space>.yml`, owned by the
person: simpler to guard, but contradicts the decided `space.yml` and splits a space's
definition); detect only at merge time (misses direct edits and new spaces); answers as events any
process may emit, acted on by anfsd (more "evented", but anything that can append to the spool
could approve; it comes back with grants across hosts).

## 5. The ui (04b)

- **An inbox panel**: `/inbox` in the nav and a count on the overview. Rows from `approval.*`
  events' data (03's pattern: an event updates its row, no list), plus `anfs inbox ls --json` at
  start and in the 5-minute reconcile (a read under the 2 s guard). A row opens `anfs inbox show
  <id> --json` (a read: `show` is a read verb for the toolkit API already).
- **Answering**: Approve and Refuse buttons post to `r.post("inbox", id, verb)`, behind the
  checks `r.post("action")` already has (the origin check, `ANFS_UI_REMOTE_ACTIONS`), and run
  `{"run":["anfs","inbox","approve",id,"--via","ui"],"yes":true}` through the toolkit API. The ui
  already holds `toolkit: write`; `approve` and `refuse` are writes, so nothing new is granted.
  A `policy` item has no buttons: it shows the command to run in a terminal (the API would answer
  77 anyway).
- **Not** in 04b: a narrower `answer` grant so the ui could drop `write` (worth it when the ui has
  a login), and OS notifications (after the ui, as decided; a small `notify:` sink later, e.g.
  `terminal-notifier`/`osascript` on macOS and `notify-send` on Linux).

*Alternatives:* the ui read-only, showing the command to paste (no write path at all; the
decided order puts answering in the ui); a new grant now (no gain while the ui has `write` for
`pcm up` and `ppm install`).

## 6. A2A

Checked against A2A 1.0.0 (below). Its task states: `submitted`, `working`, `input-required`,
`auth-required` (both "interrupted"), `completed`, `failed`, `canceled`, `rejected` (terminal).
02b's `run.*` already follows them. With 04:

| Run | When |
| --- | --- |
| `run.working` | as 02b; again (data `step`) while anfsd verifies or performs a step |
| `run.auth-required` | the harness exited 0 with commits and a step waits on the person (data `approval`, `step`). The run holds no slot |
| `run.completed` | every `land:` step `done` (data adds `landed: merge,install`), or a rule with no `land:` (as 02b) |
| `run.canceled` | a step refused (`reason: refused`, the answerer's actor), or `anfs agents stop` |
| `run.failed` | as 02b, plus `reason`: `verify`, `merge`, `install`, `push`, `expired` |

**Why `auth-required`, not `input-required`:** A2A's auth-required is an authorization the agent
cannot give itself, fulfilled out of band, and its 7.6 says the transition alone is never the
authorization; that is an approval exactly. `input-required` (the client continues the same task
with a message) is kept for a later plan: an agent asking the person a question mid-run, answered
with `anfs inbox answer <id> TEXT` and the harness resumed (`claude --resume`). Not built in 04.

*Alternatives:* the run ends `completed` when the harness exits and approvals live apart (simpler
run states, but "is r-X landed?" needs two records); `input-required` for approvals (A2A's
meaning is more input, not permission).

## 7. Retiring "cnfs must SendMessage anfs-dev"

The rule (in cnfs/partitia's Claude memory, `anfs-bugs-to-backlog.md`: "always ping after
filing", and anfs-dev "messages back when the fix is installed") has two halves:

- **cnfs → anfs-dev**: `report.filed` reaches anfs-dev's dispatch rules (02b; on when the person
  turns them on). With 04 the outcome comes to the person as an inbox item, not a chat message.
  While the rules are off, the person follows `anfs events ls -f --type report.filed` or the ui's
  events page.
- **anfs-dev → cnfs**: the merge step closes the report (`anfs report close`, by anfsd, recorded),
  the source moves (`source.updated`), and cnfs's `anfs report wait <id>` (on events since 02a)
  returns. No message.
- Withdrawn by hand after 04a is merged and the rules are on (below). The ROADMAP's Retiring line
  then says done.

## 8. Slicing, files, tests

Two plans, each merged alone:

| Plan | What | Repos |
| --- | --- | --- |
| **04a** | `approvals:`, `land:`, `verify:` in the parser; the policy file; `internal/land` (records, requests, auto, performing, expiry, restart); `run.auth-required` and the new reasons; `Denied` + `denyRead`; `anfs inbox` (ls, show, approve, refuse, request); `anfs bus status`'s inbox row; `anfs agents ls`'s status; anfs-dev's `approvals:` (all person) and its rules' `land: [merge, install, push]` and `verify:` | runtime, core, anfs-dev, docs |
| **04b** | the ui's inbox panel and answering | core (`containers/ui`), docs |

OS notifications are a later plan, after 04b.

**Files (04a).** runtime: `go/internal/space` (keys), `go/internal/land` (new), `supervise` (the
hand-over at exit, `auth-required`), `harness` (`Denied`, claude's `denyRead`), `dispatch` (policy
compare at load), `cmd/anfsd` (`anfsd inbox` plumbing that `anfs inbox` calls for validation and
the dry "what approve would run"), README, AGENTS.md. core: `packages/anfsd/home/.local/bin/anfs-inbox`
(new), `anfs-bus` (row), `anfs-agents` (status), `lib/anfs/events.sh` (`approval.*` in the
completion types), AGENTS.md ("Dispatch": landing; "`space.yml`": `approvals:`; "Events"), the
`anfs-bug-report` skill (no ping). anfs-dev: `spaces/space.yml`. docs: log, ROADMAP.

**Test spec.** Go, with real git repos in temp dirs and a fake `ppm`, `anfs` and harness
(`testfake`): a run with commits gives `approval.requested` for merge, `run.auth-required`; given
→ ff-merge of the exact head, `source.updated`, `anfs report close` called, then the install
request with the changed installed packages only; push with a fake remote; `auto` gives
requested, given by `anfsd`, done, with no wait; `merge: auto` without `verify:` waits;
a `space.yml` change in the diff forces person; `verify:` failing gives `run.failed reason=verify`
and no request; expiry: head moved, main diverged, job gone, run stopped, a newer policy; refused
gives `run.canceled`; a partial merge failure names what merged; restart mid-step; the policy file:
first sight all person, a changed file keeps the old policy and requests once, accepted → in
force; every event one valid line under 4096 bytes; idle CPU still under 0.2 s in 10 s. core bats:
`inbox ls/show` from fixture records with anfsd down; `approve` asks, `-y` answers, `ANFS_RUN`
set is refused, `policy` with `-y` or no tty is refused (77 under `ANFS_RPC`); `--through`;
`refuse --reason`; `request`; rpc: `inbox ls`/`show` reads, `approve` refused under `--grant
read`. anfs-dev: `anfsd rules` gives both rules `ok` with `land:`. 04b rspec: rows from events,
the POST's origin check and `ANFS_UI_REMOTE_ACTIONS`, no buttons on `policy`.

No test starts a real harness, touches the running bus or merges a real repo.

## By hand, for the person after 04a's merge

1. Rebuild anfsd from source and restart it; `anfs src update core anfs-dev`; `ppm install
   core/anfsd`; `wsm install anfs-dev`; `anfs bus reload` (as 02b's steps).
2. `anfs inbox ls`: empty (anfs-dev's `approvals:` is all `person`, which is what a first sight
   gives, so no policy request). `anfs bus status`: `inbox ok, 0 waiting`.
3. A dry check: `anfs agents match --type report.filed ...` now prints the `land:` steps and the
   `verify:` command.
4. Your call: turn the rules on (02b's step 5); the first real report then ends in `anfs inbox
   show`; approve merge, then install; push when you choose.
5. Withdraw the ping: delete "Always ping after filing" from
   `~/.claude/projects/-Users-roberto-spaces-cnfs-partitia/memory/anfs-bugs-to-backlog.md` (and
   the "messages back" sentence), and tell the cnfs session; tick the ROADMAP's Retiring line.
6. Check once that anfsd can push under launchd (`SSH_AUTH_SOCK` in its environment), before
   approving the first push.

## Open questions for the person

1. **May a session acting for you answer merge/install/push** (recorded as its actor), or should
   every answer need a terminal like `policy`? Recommended: sessions may, `policy` never.
2. **anfs-dev's `land:`**: `[merge, install, push]`, or stop at `install` and push by hand?
3. **`merge: auto` requires `verify:`**: agreed? And should `verify:` be required for any
   `land:`?
4. **Close at merge**: anfsd closes the report after the merge (the fix is live in the source at
   once). Or only after `install`?
5. **Should the inbox hold more than approvals** in 04 (a run that failed, a report no rule
   took)? Recommended: no, approvals only; notices come with roles.
6. 02b's open question 1 (`--setting-sources`/`--bare` for dispatched claude) is still open and
   matters more once runs lead to merges: decide it with 04a.

## Sources checked (2026-10-23, read-only)

- A2A, Latest Released Version 1.0.0 (previous 0.3.0, 0.2.6, 0.1.0):
  https://a2a-protocol.org/latest/specification/
  - **Confirmed** from the page: TaskState (4.1.3) has `input-required` and `auth-required`, each
    "an interrupted state"; completed, failed, canceled, rejected are terminal (9.4.6);
    input-required is continued by a new message on the same task (3.4.3: "the same `taskId` and
    `contextId`"; 6.3's example reuses `taskId` with a new `messageId`); auth-required (7.6):
    credentials arrive out of band (7.6.1 "Agents MUST arrange to receive credentials via an
    out-of-band means"), in-band only if negotiated (7.6.3), and "Agents MUST NOT treat the
    `TASK_STATE_AUTH_REQUIRED` state transition, by itself, as authorization" (7.6.4);
    `TaskStatusUpdateEvent` (4.2.1) is `taskId`, `contextId`, `status`, `metadata`, with no
    `final` field in 1.0.0.
  - **Not confirmed**: the page was read through a fetch tool that summarizes, in parts, so the
    quotes above are as it returned them, not checked line by line; the last quarter of the page
    was not read; nothing says whether A2A means "human approval" by auth-required (its text is
    about credentials and authorization in general). The mapping in section 6 is ours.
- The Linux Foundation hosting A2A was not looked up again (AGENT-NATIVE states it).
- Claude Code 2.1.296: the strings `denyRead`, `allowUnixSockets`, `allowAllUnixSockets`,
  `allowLocalBinding` are in its binary; what they do under `-p` was not run.
- Read, not run: core `44e456b`, runtime `ae71386`, anfs-dev's `spaces/space.yml` and `anfs-work`;
  `anfs bus status` (all ok; dispatch 0 rules loaded, 2 disabled).
