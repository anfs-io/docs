---
objective: Agents are first-class actors in anfs, not only its callers. A space is where they work; every space gets an identity, a durable event log, and later dispatch, inboxes and approvals, offered to every layer built on anfs
status: decided 2026-10-23 (AGENT-NATIVE.md, "Decided"); 01 merged 2026-10-23; 02 and 03 merged 2026-10-23; 03 accepted 2026-10-24; plan 04 written 2026-10-23 for review; plan 05 written 2026-10-23 for review; review 2026-10-24 (AGENT-NATIVE-REVIEW.md): 02d planned, 04 amended, 05 revised; 02d merged and installed 2026-10-24; 05a (events repo, local, untagged) and 05b merged and installed 2026-10-24, stream recreated in the new subject shape; plan 06 written 2026-10-24 for review
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
| `02-daemon` | The bus is **NATS with JetStream**, native on the host; 01's JSONL files are its spool (emitting never blocks or fails) and the complete, greppable record. Four plans, each merged alone ([index](agents/02-daemon/plan.md)): **[02a](agents/02-daemon/02a-bus.md)** `core/nats` (pinned nats-server and nats, `127.0.0.1:47222`, users, JetStream in STATE, launchd / systemd `--user`), `core/anfsd` with the shipper (spool to stream `ANFS`, subjects `anfs.<space id>.<noun>.<verb>`, to become `anfs.<scope>.<layer>.<noun>.<verb>` in 05b, exactly once by event id, resuming from cursors), `anfs bus`, the runtime repo with CI and pinned releases; `report wait` on events. **[02b](agents/02-daemon/02b-dispatch.md)** `access:`, `agents:` and `dispatch:` in `space.yml`; the dispatcher and supervisor (`run.*` events, A2A's task states), `anfs agents`; the report as a view (records in STATE, ids unique per space, BACKLOG.md rendered by the bus); anfs-dev's rules start the maintainer in a job on `report.filed`. **[02c](agents/02-daemon/02c-podman.md)** `podman events` into `container.*` and `pod.*`. **[02d](agents/02-daemon/02d-safe.md)** safe to turn on dispatch: a spool per run (`runs/<id>/events/`, anfsd stamps the actor, `actorclaimed`), an env allow-list, settings isolation, `queue`/`rate` hold runs (`overflow:`, `hold:`, `anfs agents redispatch`), `dispatcher: anfsd\|none`; planned 2026-10-24 after AGENT-NATIVE-REVIEW.md. `events ls` keeps reading the files |
| `03-ui` | anfs-ui on events and `podman events`, its polling a slow reconcile: the person's first acceptance test of the bus |
| `04-inbox` | Approvals as events (`approval.requested` → `.given` / `.refused` / `.expired`, then `.done` / `.failed`; the step, `merge`, `install`, `push` or `policy`, is data), tied to the run (`run.auth-required` while it waits, A2A). A rule's `land:` names the steps after a run with commits, `verify:` a check first; **anfsd performs a given step** with the fixed commands shown in the request (`merge --ff-only` of the approved shas, `ppm install` of the changed packages, `push`), and closes the report. `anfs inbox` (ls, show, approve, refuse, request; `--json`), addressed to the person. Per-space `approvals:` (`person` or `auto`) is in force only once the person accepts it at a terminal (anfsd keeps the accepted policy in STATE), so no agent changes its own; a merge touching a `space.yml` is always the person's. Two plans: 04a (CLI, anfsd), 04b (the ui's inbox panel, answering through its write grant). [Plan](agents/04-inbox/plan.md); open questions answered 2026-10-24 (as recommended) |
| `05-spec` | Our CloudEvents profile as a standalone, language-neutral spec in a new public repo (`anfs-io/events`, tagged `v1.0.0`): the event (01's table tightened: `time`, `datacontenttype`, `data` required, no `null`, an actor pattern, 4096 bytes everywhere), type namespaces per layer (`io.anfs`, `io.cnfs`, `io.lgat`) and the short type, NATS structured mode on `anfs.<scope>.<layer>.<noun>.<verb>` (the type without `io.`, one shape for every layer, anfs included; decided 2026-10-24: the dev stream is recreated), the spool contract for any language, at-least-once with dedup by id and derived ids, layer extensions (lgat's `commitment`), `anfsv` rules; examples (valid and invalid) with a manifest, a JSON Schema, `anfsd validate` and `anfs events validate`. Two slices: 05a the spec repo, 05b runtime and core conform (24 gaps). What cnfs's S9 gem and lgat's indexer get. Open questions answered 2026-10-24. [Plan](agents/05-spec/plan.md) |
| `06-coordinator` | A coordinator per space. The mechanism (06a): `wsm lease <role>` (`claim`, `renew`, `release`, `holder`, `list`; also `anfs space lease`), a lease per space id and role in STATE with a fencing claim id, the holder's pid and start time, a TTL renewed by the running listener (`listening` 90 s, `working` 2 h), takeable when expired even if the pid lives; `wsm listen` resumes from a cursor of byte offsets per month file, not id order (no gap between holders, late run and recorder lines included; at-least-once); renewals atomic (temp, fsync, rename) with back-off on failure, gathers a batch and exits; `events ls --cursor`, `--after` documented as id order; `lease.claimed`/`.released`/`.expired`. The Claude adapter (06b): policy `coordinator: claude\|none` in `space.yml`, wsm writes the space's `.claude/settings.json` (its first piece of settings ownership); SessionStart claims or names the holder, every prompt takes over a dead or expired lease, Stop and SessionStart arm the listener as `asyncRewake` hooks (no model rearming), SessionEnd releases; `ANFS_COORDINATOR=off`; dispatched runs never hold it; an inbox line in every session's prompt from anfsd's inbox summary. anfsd's OS notifications on `approval.requested` (06c), and the ui shows the holder. Planned 2026-10-24 for review. [Plan](agents/06-coordinator/plan.md) |

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
