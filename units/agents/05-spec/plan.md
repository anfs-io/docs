# Plan 05 — the spec: our CloudEvents 1.0 profile, standalone, with conformance

Written 2026-10-23 against core `44e456b` (01, 02a–c and 03 merged) and runtime `ae71386`, for
the person's review. Binding: `AGENT-NATIVE.md` ("Standards, not wheels", "Beyond the dev host",
"Plugging in"), `units/agents.md`, 01's profile table (the first version of this spec), 02's
decisions 1 and 2 (subjects, `Nats-Msg-Id`, the shipper) and 02c's mapping. Nothing below reopens
them. The plan decides where the spec lives, what it says, how conformance is checked, what the
two implementations must change, and what cnfs and lgat get from it.

## Context — read these files first

- docs: `AGENT-NATIVE.md`, `units/agents.md`, `units/agents/01-events/plan.md` ("The envelope"),
  `units/agents/02-daemon/{plan,02a-bus,02c-podman}.md` and `02c-log.md`
- core: `packages/anfs/home/.local/lib/anfs/events.sh` (`_anfs_ev_envelope`, `_anfs_emit`,
  `_anfs_ev_str`, `_anfs_ev_parse`)
- runtime: `go/internal/cloudevent` (Parse, Line, Append), `go/internal/shipper` (`batch`),
  `go/internal/bus` (`Publish`), `go/internal/podman` (`Event`, `emit`), `go/internal/space`
  (`on.type`)

## 1. Where the spec lives

**Recommendation: a new small public repo, `anfs-io/events`**, tagged `v1.0.0` when the person
approves the text. It holds only the contract: `SPEC.md`, `REGISTRY.md` (namespaces and
extensions), `schema/event.v1.json`, `examples/` with `examples/index.json`, `CHANGELOG.md`, a
CI workflow that checks the examples against the schema. No implementation code.

Why: cnfs's gem and lgat's indexer pin a tag and vendor the examples into their own tests; a tag
on a repo that holds nothing else is the version, and its history is the spec's history. The repo
is in anfs-dev's `space.yml` beside docs and runtime, so jobs work in it as usual.

*Alternatives:* **docs** (`anfs-io/docs`, already public): no new repo, but docs holds plans and
logs, tags there would mean nothing, and consumers would vendor a directory out of a repo that
changes daily. **runtime** (`spec/` beside `go/`): the Go tests read the examples directly, but
the spec's version would be tied to anfsd's releases (v0.3.0 today) and a Rails or Solana consumer
would depend on the daemon's repo. The runtime vendors the examples instead (4).

01's plan keeps its table as history, with a line pointing to the repo; `events.sh`'s and
`cloudevent.go`'s headers point to the repo, not to 01.

## 2. What the spec says

`SPEC.md` is short, normative (MUST/SHOULD, RFC 2119), language-neutral. Its sections:

### 2.1 The event

One CloudEvents 1.0 event (spec 1.0.2) in the structured JSON format. The profile tightens
CloudEvents; it never loosens it.

| Attribute | Profile | Rule |
| --- | --- | --- |
| `specversion` | required | `"1.0"` |
| `id` | required | UUIDv7 (RFC 9562), lower-case, hyphenated. Globally unique, so `id` alone (not `source`+`id`) is the dedup key. Fresh ids are random; derived ids are allowed (2.6) |
| `source` | required | `/anfs/spaces/<space id>` (`^[a-z0-9]{10}$`) or `/anfs/host`. **The same for every layer**: a space is an ANFS concept, so where an event happened does not depend on who defined its type |
| `type` | required | `io.<namespace>.<segment>(.<segment>)+`, segments `[a-z][a-z0-9-]*` (2.2) |
| `subject` | optional | non-empty string, in the type's own terms |
| `time` | **required** (01: "optional, always set") | UTC, RFC 3339, exactly milliseconds and `Z`; its milliseconds equal the id's 48-bit timestamp. When the occurrence happened as well as the producer knows it |
| `datacontenttype` | **required** (01: "always set") | `"application/json"` |
| `data` | **required** (01: `{}` when none) | a flat object: keys `^[A-Za-z_][A-Za-z0-9_]*$`, values strings |
| `dataschema` | not used in v1 | readers ignore it |
| `actor` | extension, required | who acted: `<kind>[:<name>]`, `^[a-z][a-z0-9-]*(:[^\s]+)?$`, at most 128 bytes (`claude:2e9431a0`, `shell:rob`, `podman`, `anfsd`) |
| `actorspace` | extension, optional | the space id the actor acted as a member of; only with `actorrole` |
| `actorrole` | extension, optional | the role, `^[a-z0-9][a-z0-9._-]*$`; may appear alone (a role in a space with no id, as the bash emitter writes it) |
| `spacepath` | extension, optional | the space's path when written (`anfs-dev`, `cnfs/apps`); informational only |
| `anfsv` | extension, required | integer `1` |

Global rules:
- **No `null`, anywhere** (CloudEvents allows `null` for unset; we require the attribute left out).
- Strings contain no U+0000–U+001F or U+007F–U+009F (CloudEvents' String type) in attributes;
  `data` values may contain them escaped (JSON).
- Every top-level member that is not a known attribute is an extension: its name is
  `^[a-z0-9]{1,20}$` and its value a string, integer or boolean (CloudEvents' rules made MUST).
- **At most 4096 bytes** serialized, with the newline, **everywhere** (the spool and the bus): an
  event says what changed; detail is read from the item (principle 5).
- Readers ignore attributes they do not know and data keys they do not know; they skip, and log
  once, an event whose `anfsv` they do not know.

*Alternatives:* data as any JSON value (nested objects, numbers): richer for lgat's amounts and
cnfs's payloads, but bash readers and `grep` lose, and u64 amounts are not safe JSON numbers
anyway; kept flat strings for v1 (open question 3). CloudEvents' `authcontext` extension
(`authtype`, `authid`) instead of `actor`: decided otherwise (actor and role separate,
AGENT-NATIVE "Decided"); a bridge may add `authtype`/`authid` beside ours.

### 2.2 Types: names and ownership per layer

- `io.<namespace>.…`, each namespace owned by one layer and listed in `REGISTRY.md`: `anfs`
  (anfs-io), `cnfs`, `lgat`; `rws` and `roteoh` reserved for theirs. Only the owner defines types
  in its namespace and documents them; anfs's catalog (every `io.anfs.*` type with its subject and
  data keys: `report.*`, `source.updated`, `run.*`, `container.*`, `pod.*`, `podman.*`) is in
  `REGISTRY.md`; layers keep theirs in their own repos and link them.
- Inside a namespace the owner chooses the depth: anfs uses `<noun>.<verb>`; cnfs
  `<app>.<noun>.<verb>` (`io.cnfs.partitia.report.filed`); lgat `<program>.<event>`
  (`io.lgat.escrow.payment.settled`, mapped from Anchor's `PaymentSettled` by a table, not
  mechanically). Verbs are past tense: an event is a fact.
- **The short type** drops `io.anfs.` for anfs's types and `io.` for every other namespace:
  `report.filed`, `cnfs.partitia.report.filed`. It is what `--type` filters, dispatch rules'
  `on.type` and NATS subjects (2.3) use. To keep it unambiguous, **no `io.anfs` type's first
  segment may be a registered namespace** (no `io.anfs.cnfs.*`).
- A type's data may grow (new keys); removing a key or changing its meaning needs a new type.

*Alternatives:* one namespace for everything (`io.anfs.cnfs.…`): anfs would review every layer's
types; prefixing by domain we own (`io.anfs`, `io.cnfs` read as reverse DNS of anfs.io, cnfs.io):
CloudEvents only says SHOULD; open question 2.

### 2.3 Subjects on NATS

- **The binding:** CloudEvents' NATS protocol binding, **structured mode only**: the body is the
  event (the spool line as it is), header `Content-Type: application/cloudevents+json`. Binary
  mode (`ce-*` headers) is not published; a consumer that receives one may refuse it. The
  binding says nothing about subjects; the subject scheme is ours.
- **Header `Nats-Msg-Id: <id>`** on every publish (JetStream's dedup).
- **Subject: `anfs.<scope>.<short type>`** (02's, unchanged): `anfs.tkj76y3nfx.report.filed`,
  `anfs.host.container.started`, `anfs.k3x7q2mbzp.cnfs.partitia.report.filed`. `anfs.` names the
  bus (ANFS's, which every layer uses), not the type's owner. Filters: `anfs.<id>.>` a space,
  `anfs.*.cnfs.>` a layer, `anfs.*.report.>` an anfs kind. One stream (`ANFS`, `anfs.>`) holds
  every layer; per-space permissions (`accepts:`) stay one token.
- *Alternatives:* `<namespace>.<scope>.<rest>` (`cnfs.<id>.partitia.report.filed`): symmetric, but
  a stream or stream subjects per layer, and a space's filter becomes `*.<id>.>`;
  `anfs.<scope>.<type without io.>` (`anfs.<id>.anfs.report.filed`): uniform, but every existing
  subject, the ui's filter and dispatch's filters change.

### 2.4 The spool: the way in for any language

On a host with anfs, a producer appends to the spool and anfsd ships it; it needs no NATS client.

- **Path:** `${XDG_STATE_HOME:-$HOME/.local/state}/anfs/events/<scope>/<YYYY-MM>.jsonl`; `<scope>`
  the source's space id or `host`; the month the event's `time` (UTC). The directory is created
  when missing (0755), the file 0644.
- **Finding the scope:** the `id:` (top level of `space.yml`, `^[a-z0-9]{10}$`) of the nearest
  `space.yml` at or above the producer's directory (a Rails app's root); none, or no id: the host
  scope. `spacepath` is optional for other producers.
- **One event per line**, UTF-8, `\n`-terminated, in the **canonical form**: compact JSON (no
  insignificant whitespace), `data` the last member, `"anfsv":1` as written. The bash reader
  relies on it (`_anfs_ev_parse`); Ruby's `JSON.generate`, Python's `json.dumps(…,
  separators=(",", ":"))` and Go's encoder produce it with the hash built in order.
- **Atomic append, no locks:** the whole line in **one `write(2)`** on a descriptor opened with
  `O_APPEND`, at most 4096 bytes; a local filesystem (not NFS). Per language: bash `cat >> f <<<
  "$line"` (not `printf`: macOS splits it), Go `os.OpenFile(…O_APPEND…).Write`, Ruby
  `File.open(f, File::WRONLY|File::APPEND|File::CREAT, 0o644) { _1.syswrite(line) }`, Python
  `os.write(fd, line)`.
- **Never** rewrite, truncate or delete a line; files are kept (decided in 02). An emit never fails
  its caller: a refused or failed write is a warning.
- Non-canonical lines are still CloudEvents, and anfsd ships any valid event; only the bash reader
  needs the canonical form, so `anfs events validate` reports it (as `noncanonical`).

### 2.5 Delivery semantics

- **At least once**, end to end. Dedup by `id`: JetStream drops a repeat inside its window (24 h on
  the dev host); every consumer is idempotent by `id` beyond it (records what it has handled).
- **Distinct occurrences, distinct ids.** The same fact re-sent is the same id; a new fact about
  the same thing (lgat's `finalized` after `confirmed`) is a new event with a new id, correlated
  by `subject` and data.
- **Ordering:** per scope in append order on one host; none across scopes; `id` order is time
  order to the millisecond. A consumer that needs causal order uses data (`slot`, a sequence).
- Consumers ack after handling. Append-only: no event is retracted; a correction is a new event.

### 2.6 Derived ids (re-derived events)

A producer that re-derives events from an external log (the indexer re-reading the chain after a
crash, an outbox replay) must produce the **same id** for the same occurrence, or dedup fails. Two
ways, both conformant: persist the id with the record (an outbox row), or **derive it**: a UUIDv7
whose 48-bit timestamp is the occurrence's time (`time`'s milliseconds) and whose 74 other bits
come from SHA-256 of a stable key (`io.lgat.escrow.payment.settled|<signature>|<ix>|<event
index>|finalized`). It departs from RFC 9562's SHOULD that those bits be random, knowingly (open
question 4). *Alternative:* UUIDv5/v8 for derived events: breaks "id is a UUIDv7, ordered by time".

### 2.7 Extensions: ours and the layers'

- anfs's five (2.1) are part of v1. A layer registers its own in `REGISTRY.md` (name, owner, type,
  values, which namespaces carry it) by a PR to the spec repo; adding one is a minor version of the
  spec (`v1.1.0`), never an `anfsv` bump, because v1 readers ignore unknown extensions.
- Names are unprefixed but registered, and must not take a name CloudEvents' documented extensions
  define (`authtype`, `traceparent`, `sequence`, …: checked against their list when registering).
  `traceparent`/`tracestate` (distributed tracing) are allowed as CloudEvents defines them, for
  cnfs's OpenTelemetry.
- **The worked example, lgat's `commitment`:** String, `confirmed` or `finalized` (Solana's levels;
  `processed` is not published), on `io.lgat.*` only. Money moves only on `finalized`; a
  `confirmed` that never becomes `finalized` simply has no successor.
- *Alternative:* prefixed names (`lgatcommitment`): no registry needed, collision-free, but long
  (CloudEvents' SHOULD of 20 characters) and ugly in every filter.

### 2.8 Versioning and compatibility

- `anfsv` is the envelope's version (1). The spec's tags are semver: `v1.y.z` is `anfsv` 1;
  `v2.0.0` would be `anfsv` 2.
- **No bump** for what every v1 reader already tolerates: a new type, a new data key, a registered
  extension, a new namespace. **A bump** for anything that makes a v1-valid event invalid or changes
  an attribute's meaning (nested data, a new source form, a larger size limit).
- **Deploy order on a bump:** readers and shippers first, writers last (anfsd skips and logs an
  `anfsv` it does not know, and its cursor moves on). A writer writes the lowest `anfsv` that
  expresses the event.
- The 01 → 05 tightening (`time`, `datacontenttype`, `data` required; no `null`; actor pattern) is
  **not** a bump: every line any implementation has written satisfies it (the 35 lines on this
  host, checked with jq, all do).

## 3. Conformance

- **Examples** (`examples/valid/*.jsonl`, `examples/invalid/*.jsonl`, one line each, so line-level
  rules are testable): valid: `report-filed`, `report-closed`, `source-updated`, `run-completed`
  (actor `anfsd`), `container-died` (host scope, podman), `as-role` (actorspace + actorrole),
  `role-only`, `no-subject`, `unknown-extension`, `traceparent`, `cnfs-partitia-report-filed`,
  `lgat-payment-confirmed` and `-finalized` (derived ids, `commitment`), `max-size` (exactly 4096
  bytes), `noncanonical` (valid event, whitespace: valid, flagged for the spool). Invalid, one rule
  each: missing `specversion`/`id`/`source`/`type`/`time`/`actor`/`anfsv`/`data`; `anfsv` 2;
  `specversion` "0.3"; UUIDv4 id; upper-case id; source `/x`; type without `io.`; type with an
  upper-case segment; time without ms, with an offset, not matching the id; `null` subject; empty
  subject; nested data; numeric data value; bad data key; bad extension name (`actor_role`);
  object-valued extension; bad actor; actorspace without actorrole; 4097 bytes; two lines.
- **`examples/index.json`**: per file `{file, valid, rule, schema}`, `rule` a stable code (`time.ms`,
  `size.max`), `schema` whether the JSON Schema alone catches it (size, one line, time = id's
  milliseconds and canonical form it cannot).
- **`schema/event.v1.json`**: JSON Schema draft 2020-12, `$id` the raw GitHub URL of the tag (open
  question 1 has the name): the table of 2.1, `patternProperties` `^[a-z0-9]{1,20}$` with scalar
  values for extensions and `additionalProperties: false`, the UUIDv7 pattern
  `^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`, `commitment`'s enum. The
  spec repo's CI runs a pinned `check-jsonschema` over the examples (valid pass; invalid with
  `schema: true` fail). Any language validates with its own library (Ruby `json_schemer`).
- **The runnable check: both.** `anfsd validate [FILE...|-]` (runtime, `internal/cloudevent`, the
  full profile, line-level rules included, and with a spool path the scope directory and month
  file) prints `FILE:LINE RULE message`, exit 1 on any invalid; **`anfs events validate
  [FILE...|SPEC...|--all]`** (core) is the person's and agents' form: it resolves spool files as
  `events ls` does and runs `anfsd validate` (with core/anfsd absent: an error naming it; bash does
  not re-implement the validator). **Go tests** in runtime run every example (vendored into
  `internal/cloudevent/testdata/spec/` at a tag, with `VERSION`) through `Parse` and, test-only,
  through the schema (`santhosh-tekuri/jsonschema`), asserting the manifest: the hand-written
  parser and the schema cannot drift. Core's bats run `anfs events validate` with a stub `anfsd`
  (argument passing, exit codes) and the bash emitter's output through a fixture check of the
  canonical form.
  *Alternatives:* Go tests only (no tool for cnfs's developer to check their gem's spool lines on
  a dev host); a bash validator (a second, weaker implementation of the rules).

## 4. Making the implementations conform: the gap list

Checked by reading the code (read-only); the real spool's 35 lines all conform.

**runtime `internal/cloudevent` and its users**
1. `typeRE` accepts only `io.anfs.*`: another layer's spool line is refused and never shipped.
   Accept any `io.<namespace>.…` (2.2); `Short()` and `NatsSubject()` follow 2.2/2.3.
2. `Parse` accepts `null` for string attributes and data values (`json.Unmarshal` of `null` into a
   string is no error) and an empty `subject`: refuse.
3. `time` optional with any fraction: required, exactly milliseconds, equal to the id's timestamp.
4. `data` and `datacontenttype` may be absent: required.
5. `actor`, `actorspace`, `actorrole`, `spacepath` are validated by `Line` but not by `Parse`;
   `actorspace` without `actorrole` is not refused.
6. Unknown members are not checked: extension names (`^[a-z0-9]{1,20}$`) and scalar values.
7. CloudEvents' forbidden control characters (U+007F–U+009F) in string attributes pass `Line` and
   `Parse`.
8. `anfsd validate` does not exist; the spec's examples are not in testdata (only `bash.jsonl`).
9. The shipper does not check that a line's scope directory matches its `source`: log once, ship by
   `source`.
10. `internal/space` trims only `io.anfs.` from `on.type`: rules on other namespaces use the short
   type of 2.2 (works by accident today through `Short()`; made explicit and tested).
11. Optional: podman events carry podman's own `timeNano`; `time` is when anfsd wrote the line
   (milliseconds later). Use podman's time when present (2.1: "when the occurrence happened").

**core `events.sh`**
12. `ANFS_ACTOR` is taken verbatim: check the actor pattern, warn and fall back to detection.
13. `_anfs_ev_str` escapes U+0001–U+001F but passes U+007F and C1 controls, and bytes that are not
   UTF-8 (an invalid JSON text): refuse with the usual warning (a check under a UTF-8 locale, no
   fork; to confirm in the job).
14. The reader's short type strips only `io.anfs.`: strip `io.` for other namespaces (2.2), so
   `--type 'cnfs.*'` and the SPACE/EVENT rows read right.
15. `anfs events validate` (new, 3); headers of `events.sh` and `cloudevent.go`, AGENTS.md
   "Events" and 01's plan point to the spec repo; a comment says "recorded as null" where the
   space is in fact left out.

**Not gaps:** the envelope's order and compact form, UUIDv7 generation, the 4096 limit, one
`O_APPEND` write, `Nats-Msg-Id`, the `Content-Type` header, the podman mapping's types, subjects,
actor and string data, `run.*` from the supervisor.

## 5. What cnfs and lgat need

**cnfs, to start S9 (the events gem):**
- `anfs-io/events` at `v1.0.0`: SPEC.md, the schema, the examples and the manifest to vendor
  into RSpec (`json_schemer` plus the line rules); `io.cnfs` registered.
- The spool contract (2.4) with Ruby's append, and the scope rule (the app's nearest `space.yml`
  id) for development; for production the NATS profile (2.3: subject, structured mode,
  `Nats-Msg-Id`). A Postgres store is theirs: only the envelope and the dedup-by-id semantics
  carry over.
- Delivery (2.5) for the inbound consumer (a table of handled ids), and ids persisted with the
  outbox row (2.6).
- `anfs events validate` on their dev host to check what the gem writes.
- Not needed: anfsd's code, the anfs tool suite in production, NATS in development (the spool).

**lgat's indexer:**
- `io.lgat` and the `commitment` extension registered; the type mapping from Anchor events is
  theirs (a table).
- Derived ids (2.6) so a re-index dedups; `time` = block time (seconds, `.000`), `subject` = the
  transaction signature, data flat strings (amounts as decimal strings, keys base58, `slot`,
  `ix`); `confirmed` and `finalized` as two events (2.5); actor names the producer
  (`lgat-indexer`), the signer goes in data. Large Anchor payloads reduced under 4096 bytes.

## 6. Slicing, tests, by hand, open questions

**Two slices, each its own job:**
- **05a — the spec repo** (job `agents-05a events docs`): SPEC.md, REGISTRY.md (namespaces,
  extensions, anfs's type catalog from `ANFS_EVENT_TYPES`), the schema, the examples, the
  manifest, CI with a pinned `check-jsonschema`, README; 01's pointer. Not tagged until the person
  has read SPEC.md.
- **05b — conformance** (job `agents-05b runtime core docs`, after 05a's tag): gaps 1–15,
  `anfsd validate`, the vendored examples, `anfs events validate`, docs. anfsd 0.4.0 and its pins.

**Tests:** spec CI: every valid example passes the schema, every `schema: true` invalid one fails,
every file in `examples/` is in the manifest. runtime: each example through `Parse` and the schema
agrees with the manifest; `Line` refuses each gap's case; the shipper ships an `io.cnfs` line on
`anfs.<id>.cnfs.…` and logs a scope mismatch once; `anfsd validate` output and exit codes; idle CPU
unchanged. core: the emitter refuses a bad `ANFS_ACTOR`, DEL and non-UTF-8 input with a warning and
exit 0; every emitted line passes a stub-free canonical-form check; `events ls --type 'cnfs.*'`;
`anfs events validate` with a stub `anfsd`. While working, through `anfs work run`; core's
`tests/all` once at the end.

**By hand, for the person:**
1. Before 05a: create `anfs-io/events` (public, empty) and add it to anfs-dev's `space.yml`
   `repositories:`.
2. After reading SPEC.md: merge 05a, push, `git tag v1.0.0 && git push origin v1.0.0`.
3. After 05b: runtime push and tag, the sums into `packages/anfsd/pins.yml`, `anfs src update core
   && ppm install core/anfs core/anfsd`; `anfs events validate --all` is clean.
4. Tell cnfs the tag (an inbox message, or by hand until 04).
5. The real spool holds a test line from 02a's fixture run (scope `abcde23456`, actor `shell:t`,
   2026-10-23 17:46): valid, but not a real space. No tool deletes events; removing the directory
   is the person's call.

**Open questions for the person:**
1. The repo: `anfs-io/events` (recommended) or `anfs-io/spec` (room for the inbox and work-item
   specs later)? Public from the start?
2. Namespaces `io.anfs`, `io.cnfs`, `io.lgat` read as reverse DNS: keep them whether or not we own
   those domains?
3. `data` stays a flat map of strings in `anfsv` 1 (recommended), or allow any JSON now, before
   cnfs and lgat build on it?
4. Derived UUIDv7 ids (hash bits instead of random) for re-derived events: accept?
5. The 4096-byte limit on the bus too, in production (recommended), or only in the spool?

## Sources checked (2026-10-23, read-only)

- CloudEvents 1.0.2 (latest release; main is 1.0.3-wip): https://github.com/cloudevents/spec/releases,
  https://github.com/cloudevents/spec/blob/ce%40v1.0.2/cloudevents/spec.md (attribute names
  `[a-z0-9]`, SHOULD ≤ 20; `source`+`id` unique; `type` SHOULD reverse DNS; String excludes
  U+0000–U+001F, U+007F–U+009F; Integer 32-bit)
- JSON format: https://github.com/cloudevents/spec/blob/main/cloudevents/formats/json-format.md
  (`application/cloudevents+json`; `null` MAY encode unset; JSON data stored as a JSON value)
- JSON Schema for CloudEvents (draft-07, required id/source/specversion/type, extensions
  unconstrained): https://github.com/cloudevents/spec/blob/main/cloudevents/formats/cloudevents.json
- NATS protocol binding (structured and binary, `ce-` headers, mode by `Content-Type`, headers
  from NATS 2.2, nothing about subjects):
  https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/nats-protocol-binding.md
- Documented extensions (names to avoid; `authcontext`: `authtype`, `authid`, `authclaims`):
  https://github.com/cloudevents/spec/tree/main/cloudevents/extensions,
  https://github.com/cloudevents/spec/blob/main/cloudevents/extensions/authcontext.md
- `anfs-io/docs` and `anfs-io/runtime` public, `anfs-io/events` and `anfs-io/spec` free:
  https://api.github.com/repos/anfs-io/{docs,runtime,events,spec}
