# Agent native: review and recommendations

Companion to `AGENT-NATIVE.md`. Written 2026-10-24 from a review session (Claude, claude.ai) of the
plan and of `runtime/` as it stood that day. Nothing here is decided: it is input for the person and
for the agents implementing the plan. **The code was read, not run**: no tests, no install of NATS or
`anfsd` on the reviewing machine.

## 0. Framing (agreed in the session)

- **The bus is a development-environment tool**, not production. Its purpose:
  1. bring real agent monitoring into the anfs substrate;
  2. let the layers above (cnfs, then lgat) design around a bus while they develop;
  3. one day move to real production platforms that `anfsd` neither governs nor runs.
- It must stay **flexible**, so the person and the code can both evolve it.
- anfs is a **composable toolkit** (ppm, pcm, pmm, wsm, ...) in which tools that come along
  (Pi Durable, Jev, beads, Firstmate, herdr, whatever comes next) have a natural home. A **space**
  assumes little: it hosts repos, skills and agents, and leaves dev workflows to the tools the space
  brings in. It is a lightweight successor to chorus, which is shelved.

Everything below is judged against that framing. Production-grade concerns are noted only where they
shape the *contract* that higher layers will inherit.

## 1. Summary

- The plan is sound. Durable before real time, plain files, CloudEvents, NATS, a thin daemon,
  per-space approvals and the layering rule are the right calls.
- The implementation is careful. The shipper, the spool cursors and run adoption are better than
  first drafts usually are.
- **The main risk is not the bus. It is `anfsd` accreting workflow opinions until it becomes chorus
  again.** Rule to adopt: **anfs ships mechanisms; workflows are tenants.**
- What CNFS and LGAT should design around is the **contract** (the envelope, plus the actor and
  space model), never `anfsd`. `anfsd` is the local proving ground for that contract.
- Possible tagline: *"Spaces for agents. A bus between them. Everything else is a package."*

## 2. Background: durability in layers

"Durable" means five different things. Most frameworks cover one or two, so name which one a piece
provides:

| Layer | Question | Mechanism |
|---|---|---|
| Storage | Does a committed write survive power loss? | fsync, WAL, replication |
| Message | Is an event kept when no consumer is up? | durable log or queue, ack after processing |
| Execution | Does a 40-step run resume at step 23? | journal + replay (Temporal, Restate, DBOS) or checkpoints (Pi Durable, LangGraph) |
| Conversation / state | Do the agent's context and domain data persist, consistently? | transcript and documents committed together |
| Side effects | Does a retry email the customer twice? | idempotency keys, outbox, replay-safe tools, approval gates |

Rules that follow:

- Accept at-least-once delivery and make every handler idempotent. Exactly-once end to end does not
  exist.
- Ack only after the effect is durably committed.
- Classify every tool:
  - **read-only**: replay freely;
  - **idempotent write**: replay with a key;
  - **irreversible**: gate it, never auto-replay.

Where `anfsd` sits in this table:

- **Message durability:** yes. The spool plus JetStream.
- **Run durability:** process level only. A run is adopted after an `anfsd` restart, but its
  progress is not checkpointed: a retry starts the agent from zero.
- **Conversation durability:** none. This is fine for dev.

## 3. Pi Durable (context)

Released by Earendil alongside Pi 1.0 (October 2026), experimental. It is a TypeScript **library**,
not a platform.

What it provides:

- Checkpointed agent conversations that resume in a new process after a crash.
- Several conversations per harness, each with its own model, tools, instructions and working
  directory.
- Forks with ancestry (not trees).
- Multiple clients attached to one harness, to watch and steer.
- Typed JSON documents committed atomically with transcript updates: an outbox built into the agent
  loop.

Caveats:

- Tool replay after an interruption is opt-in: a tool must declare itself replay-safe.
- Request ids stop duplicate admission, not duplicate side effects.
- Storage is memory, SQLite or JSONL, with one process owning a store at a time. The default SQLite
  settings do not guarantee the newest commits survive power loss.
- Context limits still apply; it ships a compaction example.

Relevance to anfs:

- It is a **runner**: a long-lived consumer that holds conversations. It fits the bus as a peer
  consumer, not as something `anfsd` spawns per event (as with `pi -p`).
- Use the event id as Pi's request id (free dedupe), and ack the bus message after the Pi commit.
- It is also a candidate for conversation-level durability in production agents (with Restate,
  Temporal or Inngest as alternatives for workflows).

## 4. Strategy: mechanisms, not workflows

### 4.1 Keep the bus at "just enough"

The bus core should stay at:

- **the envelope**: strict CloudEvents 1.0 plus our extensions;
- **the spool**: append one line, any language, no client library;
- **the shipper**: spool to JetStream;
- **subject and consumer conventions**: `anfs.<scope>.<type>`, durable consumers named
  `<tool>-<purpose>`;
- **an event catalog** (§4.5).

Everything else is a **tenant**: something that uses only those public conventions. That includes
`anfsd`'s own dispatcher, the report sink and the podman follower. A tenant has no more privilege
than Firstmate, beads or a Pi Durable worker would.

**Acceptance test for "flexible substrate":** turn off `anfsd`'s dispatcher for one space, let
another tool (Pi Durable or Firstmate) handle it end to end, and change no core code.

### 4.2 Work items: the bus carries facts, the tracker owns state

Primitive 1 in the plan ("work items should be records on the bus") conflicts with letting each space
pick its tracker (beads, `anfs report`, Linear, the proposed anfs tracker). Recommendation:

- The bus carries work-item **events** in a shared vocabulary: `filed`, `assigned`, `closed`,
  `available`, ...
- The **record** lives in whichever tracker the space chose. BACKLOG.md, GitHub issues and Obsidian
  stay views.
- Never make the bus the source of truth for a tool's own state. This applies to beads' git-backed
  data, Pi Durable's store and Firstmate's model alike.
- Where anfs owns the tracker (`anfs report`), add a small **projection** (for example SQLite) kept by
  a consumer, so `show` and `wait` read state cheaply instead of folding the log. Today the only
  materialization is a markdown rendering.

### 4.3 `space.yml` declares slots, not workflows

```yaml
tools:
  tracker: beads          # or anfs-report, linear
  dispatcher: firstmate   # or anfsd
  runner: pi-durable      # or claude, opencode, pi
  notes: jev
```

The space says which tool fills which role; the bus is how they talk. This makes the per-space
configurability explicit, and lets `anfsd`'s dispatcher stand down per space.

### 4.4 Integration levels; ppm packages are the integration point

Most third-party tools will never know anfs exists. A tool joins at one of three levels:

1. **Wrapped.** The tool is unaware of the bus; its package adds hooks (git hooks, pcm hooks, file
   watches) that emit events on its behalf.
2. **Emitter.** The tool appends to the spool or publishes to NATS.
3. **Participant.** The tool runs a durable consumer.

A ppm package bundles the tool, its bus adapter and its skill. Install the package, and the tool
arrives connected to the bus and teachable to agents.

| Tool | Level | Notes |
|---|---|---|
| Jev (vault scanner) | emitter | note and vault change events: principle 6 applied to Obsidian |
| beads + Firstmate | participant | beads as work-item store (or a sink); Firstmate as dispatcher for opted-in spaces |
| Pi Durable | participant | long-lived runner; event id → request id; ack after Pi commit |
| cnfs Ruby gem | emitter + participant | see §7 |

### 4.5 Event catalog

An agent landing in a space needs to know what is on the bus. Add something like
`anfs events types [space]`, listing types declared by packages plus types observed in the log, each
with its last-seen time and an example. It is cheap and pays off every time an agent orients itself.

### 4.6 Harnesses as data

Agent CLIs appear monthly. Command lines are currently compiled into Go (`internal/harness/*.go`).
Move harness definitions (argv template, permission mapping, summary extraction) into
package-provided YAML, so a new harness is a ppm package, not an `anfsd` release. Keep Go code only
for what YAML cannot express.

## 5. Implementation review (`runtime/`, 2026-10-24)

### 5.1 Strengths, keep these

- **Spool as outbox.** An emit never depends on the broker.
- **Shipper cursor discipline.** The cursor moves only over the contiguous prefix NATS acknowledged;
  `Nats-Msg-Id` is the CloudEvent id, with a 24 h duplicate window. Cursors are written by temp file,
  fsync and rename. This is correct at-least-once delivery with dedupe.
- **Runs.** Each run starts under `setsid`, and a `sh` wrapper records the exit status. After a
  restart, `anfsd` adopts surviving runs, checking pid plus start time against pid reuse. `run.*`
  states follow A2A.
- **No-spin rules.** Backoff everywhere, plus an idle-CPU test. This is the direct answer to the
  toolkit-api/06 loop.
- Dispatch dedupe by rule and event id; the ack only after the record is written; the stale, rate,
  loop and queue guards; `anfsd match` as a dry run.
- Tests: an embedded NATS, fakes for every external program, and never a real harness or podman.

### 5.2 Findings, prioritized for a dev substrate

**F1. Dispatched agents may be unable to emit events (correctness, high).**

- Claude's sandbox `allowWrite` is the job, its TMPDIR and `access:` write paths
  (`harness/claude.go`, `supervise.execute`). It does not include `$XDG_STATE_HOME/anfs/events`.
- Tools an agent runs inside the sandbox (`anfs report ...`) would have their emits blocked, and
  emits are designed never to fail loudly.
- `report.closed` and anything else an agent does could vanish without an error. That defeats the
  monitoring goal.
- Verify first. If confirmed, either:
  - add the events directory (append-only by convention) to the sandbox's write list; or
  - have runs emit through an `anfsd`-owned path, for example a per-run spool file that `anfsd`
    ingests.

**F2. The envelope is closed to other layers (flexibility, high).** `internal/cloudevent` and the
profile reject what the plan itself intends:

- `type` must match `^io\.anfs\.`, so `io.cnfs.*` and `io.lgat.*` (as AGENT-NATIVE.md proposes)
  would be logged as "not an event" and never shipped;
- `source` must be `/anfs/spaces/<id>` or `/anfs/host`;
- space ids must match `^[a-z0-9]{10}$`;
- `data` must be a flat map of string to string.

Recommendation:

- Keep the envelope check strict.
- Allow `io.<layer>.<noun>.<verb>`, per-layer sources and a documented space-id rule.
- Let `data` be any JSON object within the 4096-byte limit.
- Keep "flat strings, thin events" as a convention for `io.anfs.*`.
- Subject mapping becomes `anfs.<scope>.<layer>.<noun>.<verb>`, or a per-layer root; decide which.

**F3. Only one way in is recorded (flexibility, high).**

- Locally, tools append to the spool. In production, a Rails gem will publish to NATS (or an outbox
  relay will).
- If dev supports only the spool, the gem is built against the wrong interface.
- Allow direct NATS publishes, and add a **recorder** consumer that mirrors events not originating in
  the spool into the JSONL files, keeping "the files are the complete record" true for both paths.
- Give external tools their own NATS user with publish and subscribe rights. Today the only user is
  `anfsd`.

**F4. Dispatch is welded in (flexibility, medium).**

- Make the built-in dispatcher optional per space and/or per type (§4.3).
- Document "how to be a consumer": the naming convention, filters, ack after commit, idempotency by
  event id.

**F5. Backpressure drops events permanently (semantics, medium).**

- The `stale`, `rate`, `queue` and `loop` guards write a `run.rejected` record. Because `Has()`
  dedupes by rule and event id, that event is never dispatched again.
- A `report.filed` that arrives while the maintainer's queue is full is silently lost to dispatch.
  That contradicts the first slice's done criterion ("nobody is watching").
- Higher layers will learn whatever semantic this is. Recommendation:
  - make `queue` and `rate` **delay** by default;
  - make the policy explicit per rule;
  - add `anfs agents redispatch <event>`;
  - show rejected-but-unhandled work in `anfs inbox`.
- `stale` and `loop` may stay terminal.

**F6. Self-asserted actors (contract, low for dev).**

- Any writer of the spool can claim any `actor`, including a person approving a merge. On a
  single-user dev host this is not a threat model: **do not build signing.**
- Do record in the spec (05) that in production the **publisher attests the actor**, so cnfs and lgat
  never design around self-asserted identity.
- When 04's approvals arrive, prefer approvals that come through `anfs inbox` over a path runs cannot
  use, rather than trusting spool lines.

**F7. Credentials reach runs (hygiene, low).**

- `baseEnv` passes `anfsd`'s environment through. Deny-lists such as `Bash(git push:*)` are hints,
  easily bypassed (`sh -c`, `env git push`), and `pi` enforces nothing at all.
- Make "merge, install and push are the person's" true by **absence of credentials**: scrub
  `SSH_AUTH_SOCK`, `GH_TOKEN`, `GITHUB_TOKEN` and similar, and make sure `gh` and keychain git
  credentials cannot be reached from runs.

**F8. Smaller, fix when they bite.**

- **Per-scope token buckets don't isolate scopes.** `limiter.Wait` runs inside one batch, so a
  flooding scope stalls shipping for all scopes.
- **The sink can stall.** `MaxAckPending: 1` with nak delays up to 5 minutes means one failing event
  can hold up all report rendering for about 12 minutes.
- **A recreated consumer skips events.** When a consumer is deleted and recreated,
  `DeliverNewPolicy` skips the gap. With the `stale` guard in place, delivering from the last-known
  time would be safe.
- **Unbounded growth.** `runs/`, the supervisor's in-memory indexes and the month files grow without
  limit. This is part of open question 1 (retention).
- **Scope ran ahead of the first slice.** The podman follower and four harnesses landed before the
  report pipeline was proven: more surface to test first.

### 5.3 Contract items to reserve now (cheap now, costly later)

- `traceparent` (and `tracestate`) in the envelope: CloudEvents' distributed tracing extension (§8).
- Model and token usage on `run.completed`: model, input/output tokens, cost.
- A `v`/`anfsv` bump policy and conformance fixtures (§7.4).

## 6. Test sequence (when installed)

1. `go test -race ./...` and `bats test/e2e.bats`: embedded or temporary NATS, fakes only.
2. Install `core/nats` and `core/anfsd`; check `anfs bus status`.
3. Stop NATS, emit events, start NATS again. Expect everything shipped, no duplicates, cursors right.
4. In a scratch space, add a dispatch rule using `claude` with a small `--max-budget-usd`. File a
   report and confirm the run's own `anfs report` events reach the spool (**F1**).
5. Kill `anfsd` mid-run and restart it. Expect adoption and a correct final state.
6. Fill a role's queue and observe the overflow event (**F5**).
7. Inside a dispatched run, check `env | grep -i -e token -e ssh` (**F7**).
8. Emit an `io.cnfs.*` event to the spool and confirm whether it ships (**F2**).

## 7. CNFS and LGAT: designing around the contract

Goal: LGAT (built on cnfs engines: partitia for auth and telemetry, appwork for business lines such as
CRM, accounting and chatter) is **agent native from day 1**. Agents need not run everywhere from day 1,
but the substrate is in the design. `anfsd` facilitates this **only** as a local proving ground. CNFS
designs around the envelope and the actor and space model, and never depends on `anfsd`.

### 7.1 Rails.event is the front door, not the bus

Rails 8.1's `Rails.event` (`ActiveSupport::EventReporter`) is the emission API engines should use:

- structured name and payload;
- tags that nest, and context scoped to the request or job (fiber-scoped);
- source location on every event;
- payload filtering through `filter_parameters`;
- subscribers implementing `#emit`.

It is synchronous and in-process: **no durability and no transaction awareness.** An event notified
inside a transaction that rolls back has already been emitted.

### 7.2 The cnfs bus gem: four stages

| Stage | Where | What |
|---|---|---|
| Emit | engines | `Rails.event.notify(...)`; context carries tenant (space id), actor, role, request id |
| Map | gem subscriber | allow-list of domain events (never `sql.active_record`-style firehoses) → CloudEvent `io.cnfs.<engine>.<noun>.<verb>` |
| Persist | transactional outbox | business facts become an outbox row in the same transaction as the change; pure telemetry may skip this |
| Ship / consume | transports | out: dev → spool append or NATS; prod → outbox relay to NATS or Postgres/Solid Queue. In: events → ActiveJob, idempotent by event id (an inbox table) |

Engines know only `Rails.event` and the vocabulary; transport is configuration. `anfsd` is one dev
transport target and is absent in production.

### 7.3 Agent-native primitives for partitia (generic, every engine uses them)

1. **One actor model.** An actor is a person, an agent or the system, acting in a role within a
   tenant. Every write and every event records actor and role (the envelope's `actor`/`actorrole`).
   Staff and agents differ only in grants.
2. **Tenant = space**, with a stable id. ros-apartment lineage.
3. **Approvals engine.** An agent proposes an action; per-tenant policy decides `auto` or `person`;
   the action executes and is audited. Policy relaxes as trust grows. This is `space.yml`'s
   `approvals:` made durable in Postgres.
4. **An agent-facing surface per engine.** Engine actions exposed as tool definitions (MCP or
   similar) and checked against grants.
5. **Work items and inboxes in a shared vocabulary** (§4.2). Each engine keeps its own records.
6. **Telemetry on the same front door** (§8).

Production orchestration is ActiveJob/Solid Queue plus the approvals engine, adding a durable engine
(Pi Durable, Restate or Temporal) only if long-running agent conversations require it. `anfsd`'s runs,
dispatch rules and harnesses must **not** leak into CNFS.

### 7.4 Decide now

1. **Move the envelope spec out of anfs** into a small, layer-neutral, versioned spec repo with
   conformance fixtures (promote `internal/cloudevent/testdata`). Both `anfsd` and the Ruby gem test
   against it. This forces F2.
2. **Build the gem early against a real engine.** Wire partitia auth to the actor model, plus one
   appwork business event, to prove the envelope fits Rails while it is still cheap to change.

## 8. Worked example: chatter

chatter (cnfs/appwork) is a Turbo/Stimulus chat engine. LGAT configures instances such as:

- a **cohort group**, where a monitoring agent watches for students who need a trainer;
- a **student-to-agent DM**.

**Is the bus the way to implement this?** Partly. The line to hold: **the bus is how agents react to
chat, not how chat works.**

- **Real-time delivery to people** is Turbo Streams over Action Cable or Solid Cable, with messages in
  Postgres. Never route UI updates through the bus.
- **Reacting to what happened** is events. chatter emits `io.cnfs.chatter.message.posted` after
  commit, and who reacts is LGAT configuration. chatter knows nothing of agents.

The two configurations are different patterns:

- **Cohort monitor = observer.**
  - Subscribes to `message.posted`, tolerates seconds to minutes of delay, and mostly concludes
    "nothing to do".
  - When it acts it *proposes*: a work item in a trainer's inbox.
- **DM agent = member.**
  - The agent is a chat member (an actor) and posts through the same API as a human; replies stream
    through the same Turbo path.
  - Its turn is triggered by a `message.posted` addressed to it.
  - The transcript is chatter's message table.

Infrastructure sizing:

- **Dev:** the gem's spool transport, visible in `anfsd`.
- **LGAT production v1:** outbox plus Solid Queue, jobs keyed by chat id with concurrency limits (one
  run at a time per conversation).
- **Later:** a broker (NATS) only when several apps fan out to each other or volume demands it.

New-territory cautions:

- **Debounce per conversation.** Never run the agent on every message.
- **Idempotency and ordering per chat.**
- **Fast lane for safety.** A student in distress triggers an immediate notification to a trainer,
  never behind an approval queue or a batch window.
- **Fail open toward people.** If agents break, chat and trainers still work.
- **Consent and disclosure.** Students know that agents read cohort chats and when they are talking
  to one.
- **Audit every agent decision as an event:** actor, role, decision.
- **Per-tenant rate and cost guards.**

```
chatter (engine, knows nothing of agents)
  Message created → Turbo broadcast to members                    (real-time UI)
                  → Rails.event.notify("chatter.message.posted", chat_id:, message_id:)
                         ↓ gem: allow-list → CloudEvent → outbox row (same transaction)
LGAT configuration
  cohort chat → subscription: role "cohort-monitor" (debounced job per chat_id)
                  → concern? → work item in trainer inbox (urgent: notify now)
  DM chat     → member: agent actor "student-coach"
                  → job per message addressed to it → posts reply via chatter API
```

## 9. Telemetry and the bus

### 9.1 Not two buses: two pipelines, one front door

| | Telemetry (OTel → Grafana) | Event bus |
|---|---|---|
| Question | How is the system behaving? | What happened that someone should act on? |
| Audience | operators, dashboards, alerts | actors: agents, engines, inboxes |
| Shape | traces, spans, metrics, logs | facts (`message.posted`, `ticket.filed`) |
| Delivery | lossy is fine (sampled, batched) | must not be lost (outbox, at-least-once) |
| Retention | days to weeks | part of the record (audit, replay) |

- **Don't** drive business logic from logs or traces: sampling drops the event that mattered.
- **Don't** put spans or request metrics on the bus: high volume, nobody acts on them.
- **One emission API (`Rails.event`), two subscribers with separate allow-lists:**
  - partitia's telemetry subscriber → OTel;
  - the bus mapper → outbox.
- **Correlate** with `traceparent` in each CloudEvent, carried across the outbox and job boundary. A
  student post, the agent run and the trainer flag then appear as one trace.
- **Observe the bus and the agents with OTel**, as operational data: outbox lag, consumer pending
  counts, redeliveries, dead letters, run durations and outcomes, and model usage (tokens, cost,
  latency) per tenant and role, using OTel's generative-AI semantic conventions.
- **Derive metrics from events** (messages per cohort, concerns flagged) rather than emitting them
  twice.
- **For `anfsd` in dev:** low priority. Reserve `traceparent` and record usage on `run.completed`
  (§5.3). Dashboards later.

### 9.2 Agents reacting to telemetry: signals in, queries out

Agents do not watch raw telemetry. Alerting is the bridge:

```
app ──OTel──► Prometheus / Loki / Tempo ──► alert rule (Grafana / Alertmanager)
                                               │ fires (thresholded, grouped, deduped)
                                               ▼
                                   webhook → ops.alert.fired (one event on the bus)
                                               ▼
                                   agent queries telemetry directly (PromQL, logs, traces)
                                               ▼
                                   proposes: report.filed / rollback / page a human
```

- Detection happens where the data lives. Grouping, dedupe, inhibition and repeat intervals are
  already solved there.
- Only the conclusion crosses onto the bus: `ops.alert.fired` / `ops.alert.resolved`, with the alert
  fingerprint as the subject, so re-firing maps onto the same work item.
- The agent **pulls** detail on demand (APIs or an MCP server for the telemetry stack).
- **Periodic review** is the second flavour: a scheduled event starts an agent that looks for slow
  trends no alert catches, and files findings as reports.

Guards against circularity:

- **Agent work causing alerts that start agents:** use alert grouping, one open work item per
  fingerprint, and a rate limit per alert name.
- **Alerts about the bus or agent layer** go **out of band** to a human (email, push, chat), never
  over the bus they describe.
- **Alerts about an agent's own cost or errors** go to a human or a different role, never back to
  that agent.

Mapping:

- **Dev:** Grafana/Alertmanager webhook → a tiny receiver → `anfs emit ops.alert.fired` → a dispatch
  rule with read access to the local Prometheus and Loki.
- **Production:** Alertmanager webhook → a partitia endpoint → outbox → the same flow as chatter's
  monitor.

## 10. Prioritized actions

1. **F1:** verify, then fix, that dispatched runs can emit.
2. **F2 and F3:** open the envelope (layer namespaces, JSON `data`, direct NATS publishing with a
   recorder, a tool NATS user).
3. **F4:** make dispatch optional per space; document how to be a consumer.
4. Move the envelope spec to its own repo with conformance fixtures; reserve `traceparent` and run
   usage.
5. **F5:** backpressure policy per rule, default delay; `redispatch`.
6. Write the "mechanisms, not workflows" rule into AGENTS.md; add `tools:` slots to `space.yml`.
7. **F7:** scrub credentials from runs.
8. Event catalog (`anfs events types`).
9. Harness definitions as data (package YAML).
10. A second, non-`anfsd` consumer handling one space end to end (Pi Durable or Firstmate): the
    flexibility acceptance test.
11. cnfs: start the bus gem against partitia auth and one appwork event (`Rails.event` → outbox →
    dev spool).

## 11. Open questions added

1. Subject mapping for other layers: `anfs.<scope>.<layer>...` under one stream, or a stream per
   layer?
2. Where the layer-neutral spec lives, and who owns its versioning.
3. beads vs `anfs report`: is beads a work-item store for spaces that choose it, or a sink?
4. Do the space id format and the production tenant id format converge (one rule), or map?
5. Retention for `runs/` and the month files (extends open question 1 of AGENT-NATIVE.md).

## Sources consulted

- Pi Durable: Earendil's announcement and coverage (AlternativeTo, MindStudio, Hacker News, 36Kr),
  October 2026; `pi-agents` package page; Vercel's agent-harnesses docs for Pi.
- Durable execution landscape and costs: comparisons of Temporal, Restate, Inngest, DBOS, Trigger.dev
  (2026).
- Rails 8.1 structured event reporting: the Rails 8.1 release notes and the
  `ActiveSupport::EventReporter` API docs.
