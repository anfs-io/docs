# Agent native: what the A in ANFS means

Draft for discussion, 2026-10-23. Nothing here is decided until the person says so. It came out
of a day when the first consumer of anfs (cnfs, building partitia) filed seven reports and every
one of them went through a pipeline that was improvised by hand.

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
- **Gap:** the record lives in its rendering (a markdown section). Work items should be records
  on the bus; BACKLOG.md, GitHub issues and Obsidian are views of them.

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
4. **Local first, one host.** No external broker (NATS, Redis). Topic names leave room for a bridge
   between hosts later (RWS), but none is built now.
5. **Events say what changed.** A subscriber updates from the event, or reads one item (`show`),
   never "refresh everything".
6. **Things outside anfs come in through real event sources**, not polling: `podman events` for
   containers and pods, file watches for repos later. Polling stays only as a slow reconcile.
7. **The bus guards itself.** A client's own reads emit nothing; each subscriber is rate-limited;
   a subscription is filtered. (The 2026-10-23 loop must be impossible by construction.)
8. **Small and boring, from real cases.** Generalize only when a second client fits the first
   design: the report pipeline first, the ui second.

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

## Open questions

1. **The daemon's language.** Bash is a poor fit for a long-lived fan-out process. Ruby (as the
   ui), a small Go binary, or bash with FIFOs (simplest, fragile). Affects install, the bats
   suites, and macOS/Linux parity.
2. **The event schema**: the minimal fields, ids, versioning, and how tools emit (a `lib/anfs`
   function beside the log call?).
3. **What an agent's identity is**: a role (`anfs-dev/maintainer`) that many sessions can fill, or
   a session? Probably the role, with sessions attaching to it.
4. **Where the log lives** (STATE per the XDG rule: `~/.local/state/anfs/events/<space>/...`?),
   retention and rotation.
5. **Approvals**: which steps need the person (merge? host installs? push?), expressed as events,
   and how the person sees and answers them (the ui, a CLI, a notification).
6. **Grants across spaces**: declared where, by whom, and how cnfs's agents authenticate (the
   token model of the toolkit API extended per agent?).
7. **How this orders against setup and deploy**: spaces as agent homes ties it to setup; deploy's
   own events (r-20261022-7) would be the bus's third client.

## Roadmap impact (proposal)

- A unit `agents` (or `bus`), designed together with the setup rewrite, ahead of deploy.
- `toolkit-api`'s open questions fold into it (the daemon, grants).
- The ui's move to events becomes a plan in it, after the first slice.
