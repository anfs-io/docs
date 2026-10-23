---
objective: Agents are first-class actors in anfs, not only its callers. A space is where they work; every space gets an identity, a durable event log, and later dispatch, inboxes and approvals, offered to every layer built on anfs
status: decided 2026-10-23 (AGENT-NATIVE.md, "Decided"); 01 merged 2026-10-23; plan 02 (02a, 02b, 02c) written 2026-10-23 for review
---

The direction, the decisions and the principles are in [AGENT-NATIVE.md](../AGENT-NATIVE.md),
including "Beyond the dev host". They bind every plan here. In short: anfs is agent-compatible
today (`--json`, `-y`, skills, `anfs work`); agent native means the toolkit gives agents their
primitives (work items, events, identity and an inbox, dispatch, grants, supervision), scoped by
space and offered to the spaces built on anfs (cnfs, RWS, LGAT). The first client is the report
pipeline that ran by hand seven times on 2026-10-22/23; the second is anfs-ui.

Ahead of setup and deploy, which are redesigned around spaces as agent homes.

## Plans

| Plan | What |
| --- | --- |
| `01-events` | Space ids (`id:` in `space.yml`, `wsm id`); `lib/anfs/events.sh`: `anfs_emit`, one JSONL file per space id and month in STATE, each line one CloudEvent (1.0, structured JSON, our profile: UUIDv7 ids, `io.anfs.<noun>.<verb>` types, the `actor`/`actorspace`/`actorrole`/`spacepath` extensions); `wsm events ls` and `anfs events ls` with filters, `-f` and `--once`; `anfs report` emits `report.filed` and `report.closed`. [Plan](agents/01-events/plan.md) |
| `02-daemon` | The bus is **NATS with JetStream**, native on the host; 01's JSONL files are its spool (emitting never blocks or fails) and the complete, greppable record. Three plans, each merged alone ([index](agents/02-daemon/plan.md)): **[02a](agents/02-daemon/02a-bus.md)** `core/nats` (pinned nats-server and nats, `127.0.0.1:47222`, users, JetStream in STATE, launchd / systemd `--user`), `core/anfsd` with the shipper (spool to stream `ANFS`, subjects `anfs.<space id>.<noun>.<verb>`, exactly once by event id, resuming from cursors), `anfs bus`, the runtime repo with CI and pinned releases; `report wait` on events. **[02b](agents/02-daemon/02b-dispatch.md)** `access:`, `agents:` and `dispatch:` in `space.yml`; the dispatcher and supervisor (`run.*` events, A2A's task states), `anfs agents`; the report as a view (records in STATE, ids unique per space, BACKLOG.md rendered by the bus); anfs-dev's rules start the maintainer in a job on `report.filed`. **[02c](agents/02-daemon/02c-podman.md)** `podman events` into `container.*` and `pod.*`. `events ls` keeps reading the files |
| `03-ui` | anfs-ui on events and `podman events`, its polling a slow reconcile: the person's first acceptance test of the bus |
| `04-inbox` | Approvals as events (`merge.requested` → `merge.approved`); `anfs inbox` for the person; per-space `approvals:` (`person` or `auto`), which no agent may change |
| `05-spec` | Our CloudEvents profile as a standalone, language-neutral spec with conformance examples (valid and invalid lines), which cnfs's Rails component builds against. 01's plan holds its first version |

### Retiring today's improvisations

What the hand-run pipeline does now, and the plan that replaces it:

- The `grep` loop over `BACKLOG.md` in a session that watches for reports: a dispatch rule on
  `report.filed` (02b).
- "cnfs must SendMessage anfs-dev" (Claude Code's cross-session messages): the role's inbox
  (02, 04).
- `anfs report wait`'s git-ancestry polling: `wsm events ls -f --once --type report.closed
  --subject <id>` (01 adds the event route beside it; 02a makes `wait` follow `report.closed` and
  `source.updated` and removes the polling).
- `BACKLOG.md` stays, as a view: `ANFS_REPORT_TO` names the sink the bus renders reports into, from
  records in STATE with ids unique per space (02b).

## Not in 01

NATS, the daemon, subscriptions with cursors, dispatch, `access:`, approvals and `accepts:` are 02
and 04. 01 only writes the log and reads it, in the format 02 ships as is.
