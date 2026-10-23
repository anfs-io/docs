# Plan 01 — events: space ids, the log, `events ls`, and the report's events

Written 2026-10-23 against core main `5bbabac`. Decided with the person 2026-10-23 (the unit,
`AGENT-NATIVE.md` "Decided", and the six points of the job's brief); the event id became a
UUIDv7 the same day, and the envelope strict CloudEvents 1.0 (structured JSON, our profile), with its
spec here for other implementations; NATS with JetStream is the bus from 02, these files its spool.

## Context — read these files first

The job is `anfs work new agents-01 core docs`. Paths are core's unless they name docs.

- docs `AGENT-NATIVE.md` (the principles: durable before real time, plain files, harness-agnostic,
  local first, events say what changed, the bus guards itself) and `units/agents.md`
- `AGENTS.md`: the bootstrap header, "The Command Line", "Lists and --json", "Messages, Debugging
  and the Log", "Where files go (the XDG rule)", "wsm: spaces", "Reporting a bug", Testing
- `lib/anfs/repo.sh`: one helper in the lib, two commands (`anfs src push`, `wsm repo push`);
  events follow it
- `lib/anfs/report.sh`, `lib/anfs/table.sh`, `lib/anfs/paths.sh` (`_anfs_conf_parse`)
- wsm: `_spaces_load`, `_spec_parse`/`_spec_match`/`_specs_expand`, `cmd_new`, `cmd_list`
- `bin/anfs`'s `_rpc_events`/`_rpc_watch`: the toolkit API's watch, from `anfs.log`. Unchanged

## The envelope: our CloudEvents profile, version 1 (language-neutral)

Every event is one **CloudEvents 1.0** event in the **structured JSON format**, on one line (JSON
Lines, UTF-8, `\n`-terminated), at most 4096 bytes with its newline. The file holds exactly one
CloudEvent per line, so a shipper (02) sends each line as it is.

| Attribute | CloudEvents | Value |
| --- | --- | --- |
| `specversion` | required | `"1.0"` |
| `id` | required | A UUIDv7 (RFC 9562), lower case, hyphenated: 48 bits of Unix time in milliseconds, version `7`, 12 random bits, variant `10`, 62 random bits. Unique without coordination; ordering ids as strings orders events by time to the millisecond (ties within a millisecond are in no order) |
| `source` | required | `/anfs/spaces/<space id>`, or `/anfs/host` for the host scope. The space id is the identity |
| `type` | required | `io.anfs.<noun>.<verb>`, lower case: `io.anfs.report.filed`. The short form (`report.filed`) is what code emits and what `--type` takes; it matches `^[a-z][a-z0-9-]*(\.[a-z][a-z0-9-]*)+$` |
| `subject` | optional | What it is about, in the type's own terms (`r-20261023-1`, `acme/core#42`) |
| `time` | optional, always set | UTC, RFC 3339 with milliseconds and `Z`: `2026-10-23T16:55:23.533Z`; the same millisecond as `id` |
| `datacontenttype` | optional, always set | `"application/json"` |
| `actor` | extension, always set | Who acted: `claude:2e9431a0`, `shell:rob` |
| `actorspace` | extension | The space id of the role it acted as |
| `actorrole` | extension | The role it acted as (`maintainer`) |
| `spacepath` | extension | The space's path as wsm named it when written (`anfs-dev`, `cnfs/apps`): informational only; it changes with `wsm move`, the id never does |
| `anfsv` | extension, always set | `1`, this profile's version. A reader skips an event whose `anfsv` it does not know |
| `data` | | An object: string keys (`^[A-Za-z_][A-Za-z0-9_]*$`), string values; `{}` when none |

Rules: an optional attribute with no value is left out (never `null`); extensions are scalars with
lower-case alphanumeric names, as CloudEvents requires; nothing depends on the host or a file
path; a reader ignores attributes it does not know and lines that are not events of this profile;
no tool rewrites or deletes an event. The bash writer puts the attributes in the order above with
`data` last, and the bash reader relies on that; a reader in another language need not.

**A space id** is 10 characters from `a-z0-9` (the generator uses `a-z2-7`, as ovm's vault ids),
written once as `id:` in the space's `space.yml`, never changed, kept by `wsm move` (it travels
with the file).

Example:

```json
{"specversion":"1.0","id":"01a169b7-85e8-74aa-8809-456f41f40289","source":"/anfs/spaces/k3x7q2mbzp","type":"io.anfs.report.filed","subject":"r-20261023-1","time":"2026-10-23T17:03:17.480Z","datacontenttype":"application/json","actor":"claude:2e9431a0","spacepath":"anfs-dev","anfsv":1,"data":{"package":"core/pcm","title":"pcm run drops env","to":"file"}}
```

## Space ids

- `wsm new` writes `id: <new>` into the `space.yml` it creates. Nothing else creates a space.
- `wsm id [SPEC]`: an id spec prints that space's id (exit 1, saying `wsm id --new <space>`, when
  it has none); a prefix prints `SPACE ID` rows; no SPEC is the space around `.` (the nearest
  `space.yml` above it).
- `wsm id --new SPEC`: each space it names that has no `id:` gets one (unique among the sources'
  spaces); an existing id is never changed. `SPACE STATUS ID` rows (`new`, `kept`, `invalid id`);
  then `commit it in <source>`. The line goes before the first line that is not a comment or blank,
  so the template's header stays first; a `{}` file becomes `id: <new>`; another flow mapping is
  rewritten with yq. Through the source's file, never a link.
- `wsm list`: NOTE says `no id`, `invalid id` or `duplicate id (also <space>)`, after the counts;
  STATUS does not change. `--json` adds the key `id` (null: none).

## `lib/anfs/events.sh`

Functions only, the bash floor, no tool's code (like `repo.sh`). anfs and wsm source it.

`anfs_emit [--at DIR|--host] <type> <subject> [key=value...]` appends one event line (the short
type). The envelope is built in one function (`_anfs_ev_envelope`) and read through one accessor
(`_anfs_ev_parse`), so a change to the profile is made in one place.

- **The space**: `--host` is the host scope; `--at DIR` the nearest `space.yml` at or above DIR
  (what `anfs report` uses, so the owner space wins over the caller's); else `ANFS_SPACE` (a space
  path: `$WSM_SPACES_HOME/<path>/space.yml`, else `<sources>/<source>/spaces/<rest>/space.yml`);
  else the nearest `space.yml` at or above the current directory; else the host scope. A space
  whose path cannot be named (outside the tree and the sources) has no `spacepath`. A space with no
  valid id: a warning (`wsm id --new <path>`), and the event goes to the host scope with its
  `spacepath`.
  `WSM_SPACES_HOME` is read from the environment, else wsm's two settings files, else `~/spaces`.
- **`actor`**: `ANFS_ACTOR`; else detected: Claude Code (`CLAUDECODE=1`) is `claude:<the first
  8 characters of CLAUDE_CODE_SESSION_ID>` (`claude` without one); else `AI_AGENT` (a harness that
  sets it, e.g. `claude-code_2-1-296_agent`) is its part before the first `_`; else `shell:$USER`.
  Only Claude Code was seen here; other harnesses set `ANFS_ACTOR` until dispatch (02) does it.
- **`actorspace`, `actorrole`**: from `ANFS_AS=<space path>/<role>` (the space's id, the role); a
  role that is not `^[a-z0-9][a-z0-9._-]*$` is a warning and both are left out.
- **The file**: `${XDG_STATE_HOME:-~/.local/state}/anfs/events/<space id>/<YYYY-MM>.jsonl` (UTC
  month), `events/host/` for the host scope. The directory is made when missing.
- **Atomic, no locks**: the line is written by one `write(2)` with `O_APPEND` (`cat >> file <<<
  line`: bash's own `printf` writes through a 1 KB stdio buffer on macOS and splits a long line;
  measured). Over 4096 bytes: refused with a warning naming the size, nothing written.
- **Never fails the caller**: any error is a warning (`event <type> not recorded: ...`) and it
  returns 0. `ANFS_EVENTS=off` writes nothing (anfs.env, commented `#ANFS_EVENTS=on`; the test
  helpers set it).
- The id: milliseconds from `$EPOCHREALTIME`, the random bits from `$SRANDOM` (bash ≥ 5.1: the
  kernel's CSPRNG, `getrandom`/`arc4random`, the same source as `/dev/urandom`, and no fork).
- `anfs_space_id_new`, `anfs_space_id_read <space.yml>`, `anfs_space_id_write <space.yml> <id>`.

## `events ls`: one helper, two commands

`anfs_events_cli` declares the group (`events`, `list` alias `ls`) with the shared options;
`anfs_events_ls` runs it over the scopes the caller names (as `anfs_repo_push_cli` over what its
caller fills).

| Command | Scopes |
| --- | --- |
| `wsm events ls [SPEC...]` | the space around `.` when none; a SPEC as wsm resolves it (`_specs_expand`: an id is that space, `x/` the subtree); a space without an id is a warning |
| `anfs events ls [SPEC...] [--all]` | none, or `--all`: every scope under `events/`, the host's included; with SPECs, `wsm __event_spaces SPEC...` (plumbing) resolves them; SPECs and `--all` together are 64 |

Options: `--type GLOB` (a shell glob, short or full: `report.*`, `io.anfs.report.*`), `--subject S` (exact), `--since DUR|TIME`
(`30s`, `10m`, `2h`, `7d`; or `YYYY-MM-DD[THH:MM[:SS]][Z]`, UTC), `--by GLOB` (the actor),
`--after ID` (ids greater than it, so later in time), `-f|--follow`, `--once` (with `-f` only:
print the first match and exit 0, the backlog included, which is how an agent waits), `--json`
(the event lines as stored, one per line, not an array).

- Rows: `EVENT SUBJECT SPACE BY`; EVENT is `<time to the second>Z <type>`, SPACE the path (or the
  id; `host`), BY the actor and ` as <role>`. Oldest first. A list that is not of a source's
  resources (rule 5).
- **Follow**: one open descriptor per scope on its newest month file; each tick, new lines are read
  from it (a partial line waits for its newline); a newer month file, a new scope directory (`anfs`
  with no SPEC) or a file that did not exist yet is opened from its start. The tick is a `read -t 1`
  on a private FIFO: no process per tick, and no busy loop (a tick that returns early falls back to
  `sleep 1`). Printed as rows come (tab-separated, or JSON lines). A line written to the old month's
  file after the new month's appeared is missed (milliseconds at a month boundary; accepted).
- Completion: SPEC is wsm's `complete_spec` (anfs asks `wsm __complete`); `--type`'s kind is
  `event-type`, offering the types toolkit code emits (`report.filed`, `report.closed`, `report.*`).

## `anfs report`

- `new`, filed to a file: `anfs_emit --at <the file's directory> report.filed <id> package=<pkg>
  title=<title> to=file`. To GitHub: `--host`, `to=github repo=<owner/repo>`. `-n` emits nothing.
- `close`: `report.closed <id> commit=<sha> source=<source>`, into the same scope.
- `wait` stays as it is; its help, the skill and AGENTS.md name the event way: `wsm events ls -f
  --once --type report.closed --subject <id>`.

## What comes next (02)

The bus is NATS with JetStream (`core/nats`, native on the host), and `core/anfsd` ships these
files to it. So the files are the **spool** (emitting writes a file and never waits on the bus,
never fails because of it) and stay a **greppable mirror**. Nothing in 01 may make a line anything
but one CloudEvent ready to send as is: no framing, no local-only fields, no rewriting. The
reader `events ls` stays for a host without the daemon and for a person's shell.

## Files

- `packages/anfs/home/.local/lib/anfs/events.sh` (new); `report.sh` (the two emits)
- `packages/anfs/home/.local/bin/anfs`: sources events.sh; `events` group; `complete_spec`
- `packages/anfs/home/.config/anfs/anfs.env`: `#ANFS_EVENTS=on`
- `packages/wsm/home/.local/bin/wsm`: `id`, `events ls`, `__event_spaces`, ids in `new` and `list`
- `skills/anfs-bug-report/SKILL.md`: the event way to wait
- `AGENTS.md` (a section "Events"; wsm; rule 5's list; Lib Structure; Testing), `README.md`,
  `packages/{anfs,wsm}/README.md`
- Tests: `packages/anfs/tests/events.bats` (new), `report.bats`, `registry.bats`,
  `settings.bats`; `packages/wsm/tests/ids.bats` (new), `helper.bash` (`space_def` writes an id
  unless the body has one or `NOID=1`; `ANFS_EVENTS=off`); pcm's and pmm's helpers
  (`ANFS_EVENTS=off`)

## Test Spec

- ids: the generator's format; `wsm new` writes one; `wsm id` prints it; `--new` writes only a
  missing one, twice is the same id, never changes one; a `{}` file; `wsm move` keeps it; `wsm ls`
  NOTE for a missing and for a duplicate id, and `--json`'s `id`.
- emit: the space from `ANFS_SPACE`, from the directory, the host scope with neither, `--at`;
  `ANFS_ACTOR`, Claude Code's variables, `shell:$USER`; `ANFS_AS`; every line parses with yq and
  has the envelope's fields; ids are valid UUIDv7 and sort by time; over 4 KB refused with a
  warning and exit 0; `ANFS_EVENTS=off` writes nothing; an unwritable state directory warns and
  exits 0; 20 emits in parallel give 20 whole lines.
- ls: filters (`--type` glob, `--subject`, `--since`, `--by`, `--after`), specs (an id, a prefix,
  the space around `.`), `anfs events ls` (everything, the host scope included), `--json`, month
  files read in order.
- follow: `-f --once` exits 0 when another process appends a match (timed, under 5 s); a match
  already in the log ends it at once; a new month's file and a scope that did not exist yet are
  picked up.
- report: `report.filed` into the file's space with its data; `-n` emits nothing; `close` emits
  `report.closed`.

## Verification

- While working: `anfs work run agents-01 core -- bats <file> </dev/null`
- Once, at the end: `anfs work run agents-01 core -- packages/anfs/tests/all </dev/null`
- By hand, after merge: in `log.md`
