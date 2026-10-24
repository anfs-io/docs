# Agent native: what the A in ANFS means

Draft for discussion, 2026-10-23. Nothing here is decided until the person says so. It came out
of a day when the first consumer of anfs (cnfs, building partitia) filed seven reports and every
one of them went through a pipeline that was improvised by hand.

At a glance (what, how, who): [AGENT-NATIVE-OVERVIEW.md](AGENT-NATIVE-OVERVIEW.md).

## The claim

anfs today is a **human-native toolkit that agents can use**: CLIs with `--json`, `-y` and `-n`,
skills that teach agents the CLIs, isolated jobs (`anfs work`). That is agent-*compatible*. Any
well-built CLI suite gets there.

**Agent native** means agents are first-class actors *in* the system, not only callers of it.
Unix gave processes a few primitives (files, pipes, signals, users, cron) and everything else was
built from them. anfs should give agents theirs: work items, events, identity and an inbox,
dispatch, grants, and supervision. And because every ANFS layer serves the layers above it
(ANFS → Roteoh → RWS → LGAT, cnfs on anfs), these primitives are not anfs's internals. They are
what anfs **provides**, to every space, for that space's own agents.

## The worked example: 2026-10-22/23

cnfs's agent hit pcm bugs and missing features while building partitia. What happened, and what
each step really needed:

| Step | What we used | The primitive that was missing |
|---|---|---|
| cnfs files a bug | `anfs report`, appending markdown to `docs/BACKLOG.md` | **Work items**: structured, addressable, with a state |
| anfs-dev notices | a `grep` loop in one Claude session; later a nudge from cnfs | **Events** and **subscriptions** |
| cnfs pings anfs-dev | Claude Code's cross-session SendMessage (Claude only, this machine only) | **Agent identity and an inbox**, independent of the harness |
| a fix is started | a sub-agent in an `anfs work` job, started by hand | **Dispatch**: on an event, start an agent in an isolated job |
| the fix lands | ff-merge, install, `anfs report close`, a message to cnfs | **Completion events** (`report.closed`, `fix.available`) |
| cnfs waits for it | `anfs report wait` (polls git ancestry) | **Subscriptions** across spaces |
| all of it | worked only while one session stayed open | **Supervision**: listeners that outlive sessions |

The pipeline (report → bus → dispatcher → isolated job → merge → close → notify the reporter)
ran seven times in a day. It is the product, done by hand.

The same day showed the cost of not having events: anfs-ui polled the toolkit API, the API's own
connections came back as "something changed", and the loop pegged ten cores (toolkit-api/06,
2026-10-23). Thin events ("a command ran") plus expensive re-reads (fork a `list` pipeline) is
the shape that keeps producing those bugs.

## The primitives

For each: what it is, what exists today, the gap.

### 1. Work items

A request for work with an id, an owner space, a state (`open` → `fixed` → `available`, or
`closed`), and links to what resolved it (commits, jobs).

- **Today:** `anfs report` ids (`r-YYYYMMDD-N`), status lines in a markdown file, or GitHub
  issues. pjm and `ovm tasks` hold the person's own tasks in Obsidian.
- **Gap:** work items have no shared vocabulary on the bus, and anfs report's record lives in its
  rendering (a markdown section).
- **The bus carries facts, the tracker owns the record** (revised 2026-10-24 after
  `AGENT-NATIVE-REVIEW.md` §4.2). The bus carries work-item **facts** in a shared vocabulary
  (`filed`, `assigned`, `closed`, `available`, ...). The **record** lives in the tracker the space
  chose: `anfs report`, beads, Linear, ...; BACKLOG.md, GitHub issues and Obsidian stay views.
  The bus is never the source of truth for a tool's own state (beads' git-backed data, a Pi
  Durable store, Firstmate's model alike). Where anfs owns the tracker (`anfs report`), a consumer
  may keep a small projection (SQLite, say) so `show` and `wait` read state without folding the
  log.

### 2. Events and subscriptions (the bus)

Facts that happened, with what changed: `{space, kind, id, change, at, by}`, e.g.
`{space: anfs-dev, kind: report, id: r-20261022-2, change: filed, by: cnfs/partitia}` or
`{kind: container, id: stack/postgres, change: started}`.

- **Today:** `anfs.log` start/end lines; the toolkit API's watch stream re-emits them; nothing says
  *what* changed.
- **Gap:** typed events, emitted where the tools already log; subscriptions with filters; a
  durable log so a subscriber that was asleep catches up.

### 3. Agents: identity, presence, inbox

Who is acting: an agent has a name in a space (`anfs-dev/maintainer`, `cnfs/partitia`), a harness
(claude, opencode, pi, ...), and an inbox that anyone with a grant can post to.

- **Today:** Claude Code session names and SendMessage; nothing for the other four agent CLIs
  installed here; nothing that survives a session.
- **Gap:** an anfs-owned identity and inbox. An inbox is a subscription to events addressed to
  you.

### 4. Dispatch

A rule that starts an agent: *on `report.filed` in `anfs-dev` with `package: core/*`, start the
maintainer in a new job with this prompt*. The agent works in an isolated `anfs work` job, as
today.

- **Today:** me, reading and starting sub-agents by hand.
- **Gap:** declared rules (in `space.yml`?), a harness-agnostic launcher (`claude -p`,
  `opencode run`, ...), and the person's approvals as events too (`merge.requested` →
  `merge.approved`), so a person stays in the loop where they want to be.

### 5. Grants

What an agent may do, and where: read, write, publish to a space's topics, dispatch in it.

- **Today:** the toolkit API's `--grant read|write` and its token; pcm's `toolkit:` grant.
- **Gap:** grants per agent and per space, so cnfs's agents can file into anfs-dev and read its
  closures, and nothing more.

### 6. Supervision

Something that stays up: holds subscribers, runs dispatch rules, restarts what dies, and keeps a
record.

- **Today:** launchd starts one `anfs rpc` process per connection (inetd style). Nothing is
  long-lived except the ui.
- **Gap:** one long-lived anfs daemon per host (launchd / systemd), which is the bus and the
  supervisor.

## The layering rule

Every layer serves one or more layers above it, so each primitive is **scoped by space** and
**offered to the spaces built on it**:

- cnfs's agents get their own topics, inboxes and dispatch rules for their own work (a tenant's
  request, a failing build, a deploy), on the same bus, without knowing anfs-dev exists.
- Traffic *between* layers is just events that cross a space boundary under a grant: upstream
  (`cnfs` files a report into `anfs-dev`), downstream (`anfs-dev` publishes `report.available`,
  which cnfs subscribed to).
- RWS does the same for its tenants, LGAT for its users' agents.

This gives the **space** its meaning: it is where agents live. A space has an identity, secrets
(cnfs's r-20261022-4: a fnox provider per space), topics, an inbox, grants and dispatch rules.
**Setting up a space is creating a home for agents**, which is the question `units/setup.md`
was stuck on.

## Principles

1. **Durable before real time.** Agents sleep and sessions end. The bus is an append-only log per
   space (and topic), each subscriber with its own cursor, plus a push to whoever is connected.
   `anfs.log` is already a crude one; evolve it rather than replace it.
2. **Plain files.** JSONL that `cat`, `grep` and `jq` read. An agent with a shell can always
   inspect the bus without a client library. Views (BACKLOG.md, GitHub issues, the ui, Obsidian)
   are sinks.
3. **Harness-agnostic.** Nothing depends on one agent CLI. Dispatch launches any harness the space
   allows; inboxes are anfs's, not the harness's.
4. **Local first, standard parts.** One host first, on mature standards rather than our own wheels:
   CloudEvents for the envelope, NATS + JetStream for the bus (from 02; superseding this draft's
   first "no external broker"). Bridges between hosts (RWS) use NATS leaf nodes when needed; none
   is built now.
5. **Events say what changed.** A subscriber updates from the event, or reads one item (`show`),
   never "refresh everything".
6. **Things outside anfs come in through real event sources**, not polling: `podman events` for
   containers and pods, file watches for repos later. Polling stays only as a slow reconcile.
7. **The bus guards itself.** A client's own reads emit nothing; each subscriber is rate-limited;
   a subscription is filtered. (The 2026-10-23 loop must be impossible by construction.)
8. **Small and boring, from real cases.** Generalize only when a second client fits the first
   design: the report pipeline first, the ui second.
9. **anfs ships mechanisms; workflows are tenants** (2026-10-24, `AGENT-NATIVE-REVIEW.md` §4.1).
   The bus core is the envelope, the spool, the shipper, the subject and consumer conventions
   (`anfs.<scope>.<layer>.<noun>.<verb>`, the type without `io.`, one shape for every layer:
   decided 2026-10-24 in agents/05; durable consumers named `<tool>-<purpose>`) and the event
   catalog.
   Everything else is a tenant that uses only those public conventions: anfsd's own dispatcher,
   report sink and podman follower included, with no more privilege than Firstmate, beads or a Pi
   Durable worker would have. The risk this guards against is anfsd accreting workflow opinions
   until it is chorus again. **Acceptance test:** turn off anfsd's dispatcher for one space and
   let another tool handle that space end to end, with no change to core.

## How tools join

Most tools will never know anfs exists. A tool joins the bus at one of three levels:

1. **Wrapped**: the tool is unaware of the bus; its package adds hooks (git hooks, pcm hooks,
   file watches) that emit events on its behalf.
2. **Emitter**: the tool appends to the spool or publishes to NATS.
3. **Participant**: the tool runs a durable consumer.

A **ppm package bundles the tool, its bus adapter and its skill**: install it, and the tool
arrives connected to the bus and teachable to agents. The review's table places Jev, beads,
Firstmate, Pi Durable and cnfs's gem (§4.4).

**Slots, a direction only.** A space may one day say which tool fills which role:

```yaml
tools:                    # direction, not config: nothing reads this key
  tracker: anfs-report    # or beads, linear
  dispatcher: anfsd       # or firstmate
  runner: claude          # or pi-durable, opencode, pi
  notes: jev
```

Not built and not a `space.yml` key. Today there is one switch: a per-space "built-in dispatcher
off" (agents/02d), which is what the acceptance test of principle 9 needs.

## What exists, placed in this picture

| Piece | Becomes |
|---|---|
| `anfs.log` | the seed of the event log |
| toolkit API (`anfs rpc`, watch, grants, token) | the bus's client protocol and its grants |
| `anfs work` jobs | where dispatched agents work (unchanged) |
| `anfs report` (ids, close, status, wait) | the first work-item type; publishes events, BACKLOG.md and GitHub become sinks |
| anfs-ui | the bus's second client (moves off polling) |
| pjm, `ovm tasks` | work items from the person's notes; later publishers |
| skills (`anfs-bug-report`, ...) | how agents learn the protocol |
| `space.yml` | where a space declares its agents, dispatch rules and grants |

## A first slice

End to end, replacing today's improvisation and nothing more:

1. `anfs report` publishes `report.filed` to the owner space's log (BACKLOG.md still written, as a
   sink).
2. A dispatch rule in anfs-dev's `space.yml` starts the maintainer agent in a job on
   `report.filed` (or, at first, posts to its inbox and lets a running session act).
3. `anfs report close` publishes `report.closed`; installing the fix publishes `report.available`.
4. cnfs's space subscribes to its own reports' events; `anfs report wait` becomes a subscription.
5. The daemon (6) holds it together; the ui is not moved yet.

Done when: cnfs files a report, nobody is watching, and the fix's availability reaches cnfs's
inbox with no human or session in the middle except where an approval is declared.

## Decided (the person, 2026-10-23)

- **A space is where agents work together; a human is invited in.** The direction of travel of
  chorus ("inviting agents into the space"), now in anfs. The space is the unit of everything
  above: identity, topics, inbox, grants, dispatch, approvals.
- **Actor and role are separate, but only in the event log for now.** Every event records who
  acted and as what in separate fields (CloudEvents extensions `actor`, `actorspace`, `actorrole`;
  e.g. `claude:5a52` as `maintainer` of anfs-dev's id), because
  history can't be fixed by hand later and config can. Everything else waits (YAGNI): addresses
  such as `anfs-dev/maintainer` are used by dispatch and inboxes and simply turn out to be roles
  later.
- **Approvals are per-space policy, in `space.yml`, and evolve as trust grows**:

  ```yaml
  approvals:          # who must approve: person, or auto
    merge: person
    install: person
    push: person
  ```

  anfs-dev starts with all three `person`. A space that needs less hand-holding moves steps to
  `auto` over time. (When roles arrive, the values become role names; a by-hand edit.) **No agent can change its own space's `approvals:`**: editing the policy always
  needs the person, whatever the policy says.

- **Spaces have stable ids** (`id:` in `space.yml`, never changed, kept by `wsm move`). The event
  log is stored by id (`~/.local/state/anfs/events/<space-id>/YYYY-MM.jsonl`) so history survives
  renames; events record the id and the path at the time, and `by.as` records `{space: <id>, role}`.
- **Events: one lib, two commands**, like `repo.sh`: `wsm events ls [spec]` (the space you are in;
  `x/` for a subtree) and `anfs events ls` (everything, plus host-level events); `-f`, and
  `--once` for an agent waiting on an event.
- **Approvals are answered** first by the CLI (`anfs inbox`), then anfs-ui, then OS notifications
  (the in-house, cross-platform path first).
- **`access:` in `space.yml`** declares what actors in the space may reach beyond it (`{path, mode:
  read|write}`); dispatch hands it to the harness (`--add-dir`, allow/deny rules). `links:` stays,
  for people. In agents/02.
- **Cross-space permissions (`accepts:`) are on the roadmap**: config and enforcement, no history
  change; events already record the origin (`by`) and the destination (`space`). Until then, open.
- **The daemon is Go** (`anfsd`), in `anfs-io/runtime` under `go/` (one module, `cmd/<program>`,
  room for more programs). Users get prebuilt binaries pinned by sha256; developers get Go from
  `stack/go` through mise. The daemon stays thin: tools stay bash and emit through
  `lib/anfs/events.sh`.

- **Standards, not wheels (2026-10-23)**: the envelope is **strict CloudEvents 1.0** (structured
  JSON; `id` UUIDv7, `source` `/anfs/spaces/<space-id>`, `type` `io.anfs.<noun>.<verb>`;
  extensions `actor`, `actorspace`, `actorrole`, `spacepath`, `anfsv`), and from 02 the bus is
  **NATS + JetStream** (subjects by space and type, durable consumers, accounts and permissions,
  leaf nodes for bridges), on the dev host as in production. The JSONL files are the spool (an
  emit never blocks or fails) and a greppable mirror; `anfsd` ships, dispatches and supervises,
  and is not a broker. Production agent orchestration uses a durable workflow engine (Temporal,
  Restate or Inngest) through a cnfs component; A2A's task states are the reference for work items
  and inboxes (to be checked before 03). What stays ours: spaces as agent homes, identity and
  roles, approvals policy, the layering rule, the toolkit integration.

## Beyond the dev host: the bus in production

cnfs is a factory for business apps; in production those apps want to be agent native too, but
they will not run the anfs tool suite. lgat builds on cnfs components. So what goes into the world
is the **protocol**, not anfs:

1. **The envelope is the contract**: strict CloudEvents 1.0 with our extensions (`actor`, `actorspace`, `actorrole`, space ids),
   type naming, and semantics (append-only, at-least-once), versioned (`v: 1`), written as a
   short language-neutral spec with conformance examples (`agents/05-spec`). Nothing in an event
   depends on a host or a filesystem: ids are UUIDv7, a space's identity is its id (its path is
   informational).
2. **Implementations per environment**: on a dev host, anfs (bash emitters, JSONL, `anfsd`), the
   reference implementation. In a production Rails app, a **cnfs component** (a gem/engine) that
   implements the same envelope on the app's own infrastructure (Postgres: an events table with
   Solid Queue or `LISTEN/NOTIFY`; NATS at a larger scale, the contract unchanged). lgat includes
   that component like any other; its Solana programs' on-chain logs become envelope events through
   an indexer.
3. **Bridges between buses**: an adapter subscribes on one bus and republishes on another, under
   a grant (`accepts:`). A production error in an lgat app becomes `report.filed` in cnfs's dev
   space, cnfs's dispatcher starts an agent, the fix flows back as a release; a toolkit fault goes
   upstream to anfs-dev, as cnfs's reports did by hand on 2026-10-22. Every layer is an
   agent-native factory for the layer above, joined by one event contract.

Spaces carry over: a tenant or organization in a production app is a space, with a stable id,
actors (its users and its agents), roles and approvals: the same model on a different store.

Nothing in production is built now. What is done now is keeping it possible: the portable
envelope in 01 and the spec in 05. The Rails component belongs on cnfs's roadmap, built against
our spec: the first test of "each layer serves the one above".

### Plugging in: Rails apps (cnfs) and Solana programs (lgat)

Each side gets a small adapter that turns its native events into our CloudEvents. On a dev host
the adapter appends to the anfs spool (`anfsd` ships it to NATS); in production the same adapter
publishes to NATS (or Postgres). Nothing else changes: the envelope is the contract.

- **Rails (cnfs), a gem on cnfs's roadmap, built against 05's spec.** Outbound: a subscriber to
  ActiveSupport::Notifications (or Rails 8.1's structured `Rails.event`, to be compared first)
  forwards an allow-list of the app's domain events, never the firehose (`sql.active_record`),
  e.g. `filed.report.partitia` → `io.cnfs.partitia.report.filed`, payload reduced to allow-listed
  keys; cheap in the request (one spool append in dev, a background job in prod). Inbound: a
  consumer process (a worker, or Solid Queue) subscribes on NATS and turns events into jobs or
  local notifications. The app's directory is in a space, so its events carry that space's id.
- **Solana (lgat), an indexer on lgat's roadmap.** In development `solana-test-validator` (or
  surfpool) and the indexer run as pcm services; the indexer follows program logs, decodes Anchor
  `emit!` events with the IDL, and publishes `io.lgat.<program>.<event>` with the transaction
  signature as subject and the commitment level as an extension (`confirmed` / `finalized`;
  money moves only on `finalized`, since unfinalized events can roll back). Rails does not send
  events to the chain, it sends transactions: `payment.requested` → a signer (later an agent role
  under an approval policy) submits → the program emits → the indexer publishes
  `payment.settled` → Rails consumes it.
- **What anfs provides**: the spec (05) with type namespaces per layer (`io.anfs.*`,
  `io.cnfs.*`, `io.lgat.*`, each owned by its layer), the spool format as a supported way in for
  any language without a NATS client, the bus, and pcm definitions (e.g. the validator).

## Later: roles, members, departments (on the roadmap, not built)

The full model, for when a space has more than one person, or agents that need different
authority:

- **Actor**: a person (global, one identity across spaces) or an agent instance (a session, a
  dispatched run).
- **Role**: a bundle of skills, grants, the events it handles and the approvals it may give,
  defined once by the layer that owns it (`anfs`: maintainer, operator, reviewer; `lgat`:
  marketing, finance, ...).
- **Assignment**: an actor holds roles in a space (`members:` in `space.yml`) and acts as one of
  them at a time.
- Approvals name a role and optionally an actor kind (`push: {role: lead, actor: person}`);
  inboxes are addressed to a role (a queue) or an actor (a direct message); departments are
  child spaces (`lgat/marketing`) with their own roles; no actor changes the membership or
  approvals of a space it acts in unless a role allows it.

## Open questions

1. **Retention** of the event log (kept by default; a cap later?), and of `runs/` and anfsd's
   own records, which grow without limit too (review §5.2 F8).
2. **How a role authenticates** once there is more than one host (tokens per role?).
3. **beads vs `anfs report`**: is beads a work-item store for the spaces that choose it, or a
   sink of `anfs report`'s facts? (Review §11.)
4. **Space ids and production tenant ids**: one rule for both, or a mapping? (Review §11; 05
   keeps `^[a-z0-9]{10}$` for spaces.)

The review's other new questions are taken up in agents/05: the subject mapping for other
layers and where the spec lives and is versioned.

## Roadmap impact (proposal)

- A unit `agents` (or `bus`), designed together with the setup rewrite, ahead of deploy.
- `toolkit-api`'s open questions fold into it (the daemon, grants).
- The ui's move to events becomes a plan in it, after the first slice.
