---
objective: Agents are first-class actors in anfs, not only its callers. A space is where they work; every space gets an identity, a durable event log, and later dispatch, inboxes and approvals, offered to every layer built on anfs
status: decided 2026-10-23 (AGENT-NATIVE.md, "Decided"); plan 01 written 2026-10-23 (job agents-01)
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
| `02-daemon` | The bus is **NATS with JetStream**, and 01's JSONL files become its spool (emitting never blocks or fails) and a greppable mirror. **`core/nats`** installs `nats-server` and the `nats` CLI as pinned release binaries, listening on localhost, the JetStream store in STATE (`~/.local/state/nats/`), its config in `~/.config/nats/`, run by launchd or systemd `--user`: natively on the host, not as a pcm container, because the bus must outlive the podman machine. **`core/anfsd`** is the Go daemon (prebuilt, pinned; depends on core/nats): the shipper (spool to JetStream), the dispatcher (rules in `space.yml` that start a harness in an `anfs work` job) and the supervisor, not a broker. Subscriptions with cursors are JetStream consumers; rate limits; `access:` given to the harness at dispatch. Needs `stack/go` |
| `03-ui` | anfs-ui on events and `podman events`, its polling a slow reconcile: the person's first acceptance test of the bus |
| `04-inbox` | Approvals as events (`merge.requested` → `merge.approved`); `anfs inbox` for the person; per-space `approvals:` (`person` or `auto`), which no agent may change |
| `05-spec` | Our CloudEvents profile as a standalone, language-neutral spec with conformance examples (valid and invalid lines), which cnfs's Rails component builds against. 01's plan holds its first version |

### Retiring today's improvisations

What the hand-run pipeline does now, and the plan that replaces it:

- The `grep` loop over `BACKLOG.md` in a session that watches for reports: a dispatch rule on
  `report.filed` (02).
- "cnfs must SendMessage anfs-dev" (Claude Code's cross-session messages): the role's inbox
  (02, 04).
- `anfs report wait`'s git-ancestry polling: `wsm events ls -f --once --type report.closed
  --subject <id>` (01 adds the event route beside it; 02 removes the polling).
- `BACKLOG.md` stays, as a view: `ANFS_REPORT_TO` names the sink the events are rendered into
  (02).

## Not in 01

NATS, the daemon, subscriptions with cursors, dispatch, `access:`, approvals and `accepts:` are 02
and 04. 01 only writes the log and reads it, in the format 02 ships as is.
