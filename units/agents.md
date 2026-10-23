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
| `01-events` | Space ids (`id:` in `space.yml`, `wsm id`); `lib/anfs/events.sh`: `anfs_emit`, one JSONL log per space id in STATE, the envelope (UUIDv7 ids, CloudEvents-named fields, `by: {actor, as}`); `wsm events ls` and `anfs events ls` with filters, `-f` and `--once`; `anfs report` emits `report.filed` and `report.closed`. [Plan](agents/01-events/plan.md) |
| `02-daemon` | `anfsd` in Go (`anfs-io/runtime`, `go/`; prebuilt binaries, pinned): tails the logs, subscriptions with cursors and push, dispatch rules (in `space.yml`) that start a harness in an `anfs work` job, rate limits; `access:` in `space.yml` given to the harness at dispatch. Needs `stack/go` |
| `03-inbox` | Approvals as events (`merge.requested` → `merge.approved`); `anfs inbox` for the person; per-space `approvals:` (`person` or `auto`), which no agent may change |
| `04-ui` | anfs-ui on events and `podman events`; its polling becomes a slow reconcile |
| `05-spec` | The envelope as a standalone, language-neutral spec with conformance examples (valid and invalid lines), which cnfs's Rails component builds against. 01's plan holds the first version of it |

### Retiring today's improvisations

What the hand-run pipeline does now, and the plan that replaces it:

- The `grep` loop over `BACKLOG.md` in a session that watches for reports: a dispatch rule on
  `report.filed` (02).
- "cnfs must SendMessage anfs-dev" (Claude Code's cross-session messages): the role's inbox
  (02, 03).
- `anfs report wait`'s git-ancestry polling: `wsm events ls -f --once --type report.closed
  --subject <id>` (01 adds the event route beside it; 02 removes the polling).
- `BACKLOG.md` stays, as a view: `ANFS_REPORT_TO` names the sink the events are rendered into
  (02).

## Not in 01

The daemon, subscriptions with cursors, dispatch, `access:`, approvals and `accepts:` are 02 and
03. 01 only writes the log and reads it.
