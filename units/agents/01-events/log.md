---
plan: agents/01-events
job: agents-01
date: 2026-10-23
---

# Log: agents/01-events

## What was built

- **Space ids** (wsm): `id:` in `space.yml`, 10 of `a-z0-9` (generated from `a-z2-7`, as ovm's
  vault ids, by `anfs_space_id_new` in `lib/anfs/events.sh`). `wsm new` writes one (line 2, after
  the header comment). `wsm id [SPEC]` prints it (no SPEC: the space around `.`; a prefix: `SPACE
  ID` rows); `wsm id --new SPEC` writes one into each space without, never changes one (`SPACE
  STATUS ID`: `new`, `kept`, `invalid id`), then says to commit in the source. `wsm ls`: NOTE adds
  `no id`, `invalid id`, `duplicate id (also <space>)` after the counts, STATUS unchanged;
  `--json` adds `id`. `wsm move` keeps the id (it moves the file).
- **`lib/anfs/events.sh`**: `anfs_emit [--at DIR|--host] <type> <subject> [key=value...]`
  appends one CloudEvent 1.0 (structured JSON, our profile `anfsv: 1`) to
  `$XDG_STATE_HOME/anfs/events/<space id>/<YYYY-MM>.jsonl` (`host/` for the host scope). The
  envelope is built by `_anfs_ev_envelope` only and read by `_anfs_ev_parse` only. UUIDv7 ids;
  `io.anfs.<noun>.<verb>`; `actor`, `actorspace`, `actorrole`, `spacepath`; empty attributes left
  out. One `write(2)` with `O_APPEND`, at most 4096 bytes, never fails the caller,
  `ANFS_EVENTS=off`.
- **`events ls`**: `anfs_events_cli` + `anfs_events_ls`, used by `wsm events ls [SPEC...]` (the
  space around `.`, or SPECs through `_specs_expand`) and `anfs events ls [SPEC...] [--all]`
  (everything with the host scope; SPECs through `wsm __event_spaces`). `--type` (short or full
  glob), `--subject`, `--since`, `--by`, `--after`, `--json` (the lines), `-f`, `--once`.
  `EVENT SUBJECT SPACE BY` rows. Completion: `--type` (`complete_event_type`), SPEC (wsm's; anfs
  asks `wsm __complete`).
- **`anfs report`**: `report.filed` (package, title, `to`, and `repo` for GitHub) and
  `report.closed` (commit, source), into the report file's space, or the host scope for GitHub.
  `-n` emits nothing. `report wait` unchanged; its help, the skill, AGENTS.md and README name
  `wsm events ls -f --once --type report.closed --subject <id>`.
- `anfs.env`: `#ANFS_EVENTS=on`. Test helpers (wsm, pcm, pmm, report.bats) set `ANFS_EVENTS=off`;
  wsm's `space_def` writes an id unless `NOID=1`.
- Docs: AGENTS.md "Events" (and wsm, rule 5, the keys table, Lib Structure, Testing), README
  "Events", `packages/{anfs,wsm}/README.md`, the skill.
- `anfs rpc`'s `watch` is untouched (still from `anfs.log`).

## Commits (branch agents-01, not pushed)

| Repo | Commit | What |
| --- | --- | --- |
| docs | `6a08a29` | `units/agents.md`, the plan |
| docs | `3a94615` | the plan as a CloudEvents profile; the unit's order (02 daemon and NATS in core, 03 ui, 04 inbox, 05 spec) |
| core | `870eece` | all of the code, tests and docs above (the hook bumped pcm to 0.4.1 and pmm to 0.1.6 for their test helpers; wsm 0.12.0 by hand in the script) |
| docs | (this) | this log |

## Decisions made during the work

1. **Changed while building, from the coordinator**: the id is a UUIDv7, and the envelope is
   strict CloudEvents 1.0 (person's decision), not the `{space, by, v}` shape first planned. The
   plan holds the profile as a language-neutral table.
2. **UUIDv7 from `$EPOCHREALTIME` and `$SRANDOM`**, not `/dev/urandom` through od/xxd and perl for
   milliseconds as suggested: the bash floor (5.2) has both, `SRANDOM` is the kernel's CSPRNG
   (`getrandom`/`arc4random`, the same source as `/dev/urandom`), and nothing forks. Tests check
   the version and variant bits, that the first 48 bits are the event's milliseconds, and that ids
   sort by time.
3. **`cat >> file <<< line`, a fork per event**: measured, bash's `printf >>` splits lines over
   1 KB on macOS (stdio's 1024-byte buffer) and 20 writers interleave; `cat` writes the line in
   one call (0 torn lines of 2000 at 4090 bytes).
4. **The host scope for GitHub reports**: they are filed in no space. Their `report.filed` carries
   `to=github` and `repo`. So an agent waiting on a GitHub report uses `anfs events ls`.
5. **`--at DIR` beats `ANFS_SPACE`**: a report's events go to the space it is filed in, whoever
   files it. `--host` exists for the GitHub case.
6. **A space without an id**: the event still goes out, to the host scope with `spacepath`, and a
   warning says `wsm id --new <path>`.
7. **Actor detection**: Claude Code only (`CLAUDECODE`, `CLAUDE_CODE_SESSION_ID`, first 8
   characters), plus the generic `AI_AGENT` (its first part, `claude-code`), both seen on this
   host; nothing guessed for opencode, codex or pi, which set `ANFS_ACTOR` until dispatch.
8. **`wsm ls`**: id problems are NOTE only; STATUS stays `ok`/`partial`, so no reader of STATUS
   changes.
9. **`anfs events ls SPEC`** resolves through wsm's plumbing (`wsm __event_spaces`), so specs mean
   exactly what wsm means; `--all` is what no SPEC means, and both together are 64.
10. **Follow**: one descriptor per scope on its newest month file; a newer file is drained from
    the old one, then read from its start. A line written to the old month's file after the new
    one exists is missed (accepted; documented). The tick is `read -t 1` on a private FIFO; a tick
    that returns in under 0.9 s falls back to `sleep 1`. Measured: no CPU while waiting (the test
    holds it under 0.3 s of CPU). Rows under `-f` are tab-separated (no alignment while
    streaming).
11. **`--once` counts the backlog**: an event already logged ends it at once, so waiting for a
    close that already happened does not hang.
12. **`spacepath`** is resolved from the tree (`WSM_SPACES_HOME`, from the environment or wsm's
    settings files), a source checkout, or a local source's link (a few `cd -P` forks, only when
    the cheap paths fail).

## Tests (all through `anfs work run agents-01 core …`, `</dev/null`)

- `packages/anfs/tests/events.bats` (new): 17. Ids (format, write, read); emit (every attribute,
  the month file, the tree / `ANFS_SPACE` / `--at` / `--host` / host, a space without an id, the
  actor and `ANFS_AS`, UUIDv7 validity and order, 4 KB refused, bad type or pair, off, an
  unwritable state, 20 parallel near-4 KB emits whole); ls (every scope across months, `--json`,
  aligned header, each filter, specs through wsm, 64s, completion); follow (`--once` ends within
  4 s of another process's append, and at once on an existing match; a new month's file and a new
  scope picked up; CPU under 0.3 s).
- `packages/wsm/tests/ids.bats` (new): 7. `wsm new`'s id; `wsm id` forms; `--new` idempotent and
  never changing an id; `{}` and a malformed id; `wsm ls` notes and `--json id`; `wsm events ls`
  (around `.`, an id, a prefix, no id, `__event_spaces`); `-f --once` in wsm.
- `move.bats`: ids kept across `wsm move`. `report.bats`: 2 new (file report events into the
  file's space, `-n` none, close; GitHub into the host scope, off). `registry.bats`: `EVENT
  SUBJECT SPACE BY` among the lists of rule 5. `install.bats`, `manage.bats`: two assertions
  updated for the id line and the `id` key.
- **The full run, once**: `packages/anfs/tests/all`: 691 passed, 0 failed, 50 files (pmm and pcm
  off), 54 s.

## Not run

- pmm's and pcm's suites are off (ROADMAP, 2026-10-19); only their helpers changed (one variable).
- Nothing installed on this host; the real `~/.local/state/anfs/events` was never written. The
  only command run against the real HOME was a read-only `wsm ls` to list the spaces below.

## By hand, for the person after merge

1. Reinstall core/anfs and core/wsm (`lib/anfs` is one link, so `anfs src update` brings
   `events.sh`; the scripts are linked too):

   ```bash
   anfs src update core
   ppm install core/anfs core/wsm   # never -r for core/anfs (AGENTS.md)
   wsm --version                    # 0.12.0
   ```

2. Give every existing space its id (they are written into the sources' `space.yml`; commit them
   in each source). The spaces on this host on 2026-10-23 (`wsm ls`):

   ```bash
   wsm id --new anfs-dev/      # anfs-dev, anfs-dev/orig
   wsm id --new anfs-admin
   wsm id --new lgat/          # lgat/next, lgat/orig, lgat/tech
   wsm id --new roteoh
   wsm id --new cnfs/          # cnfs/apps, cnfs/appwork, cnfs/orig, cnfs/partitia
   wsm id --new rws
   wsm id --new user/          # user/evals
   wsm ls                      # no "no id" and no "duplicate id" left in NOTE
   ```

   Then commit the `space.yml` changes in each source (anfs-dev, anfs-admin, lgat, roteoh, cnfs,
   rws, user) and push as you choose.

3. Check: `anfs report -n --title "events check" --actual x` files and emits nothing (`anfs events
   ls` shows no new event). A real file report, if you want one (your call: it appends to
   BACKLOG.md): it shows as `report.filed` in `wsm events ls anfs-dev`; `anfs report close <id>
   --commit <sha>` adds `report.closed`. Delete the section by hand afterwards; events are never
   deleted by a tool, and the two lines can be left or removed by hand.

## Open questions

1. `anfs rpc`'s grant counts `events list` as a read; `-f` through the API would run until the
   request's timeout (60 s). The API's own event stream is 02's.
2. A GitHub report's events are in the host scope; whether a GitHub-filed report should go to the
   reporting space instead is for 02, where the report's owner space is decided again.
