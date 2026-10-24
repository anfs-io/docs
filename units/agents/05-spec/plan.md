# Plan 05 — the spec: our CloudEvents 1.0 profile, standalone, with conformance

Written 2026-10-23 against core `44e456b` (01, 02a–c and 03 merged) and runtime `ae71386`, for
the person's review. **Revised 2026-10-24** after `AGENT-NATIVE-REVIEW.md` (F2, F3, F6, §5.3, §7,
§9.1, §11) and the person's decisions on it (reviewed with Claude Desktop): the envelope opens to
the other layers (any `io.<layer>`, per-layer sources, a documented space-id rule, `data` any JSON
object), direct NATS publishing becomes a first-class way in beside the spool (a recorder keeps the
files complete; a publisher user per layer), production attests the actor (`actorclaimed`),
`traceparent`/`tracestate` and run usage are reserved, and the conformance fixtures live in the
spec repo. Code and spool re-checked at the same commits.

Binding: `AGENT-NATIVE.md` ("Standards, not wheels", "Beyond the dev host", "Plugging in"),
`units/agents.md`, 01's profile table (the first version of this spec), 02's decisions 1 and 2
(subjects, `Nats-Msg-Id`, the shipper), 02c's mapping, and the decisions above. Nothing below
reopens them. The plan decides where the spec lives, what it says, how conformance is checked,
what the two implementations must change, and what cnfs and lgat get from it.

## Context — read these files first

- docs: `AGENT-NATIVE.md`, `AGENT-NATIVE-REVIEW.md` (F2, F3, F6, §5.3, §7, §9.1, §11),
  `units/agents.md`, `units/agents/01-events/plan.md` ("The envelope"),
  `units/agents/02-daemon/{plan,02a-bus,02c-podman}.md` and `02c-log.md`
- core: `packages/anfs/home/.local/lib/anfs/events.sh` (`_anfs_ev_envelope`, `_anfs_emit`,
  `_anfs_ev_str`, `_anfs_ev_parse`), `packages/anfs/home/.local/lib/anfs/bus.sh`
  (`anfs_bus_nats_users`, the server config's users and permissions)
- runtime: `go/internal/cloudevent` (Parse, Line, Append), `go/internal/shipper` (`batch`),
  `go/internal/bus` (`Publish`, `EnsureStream`), `go/internal/spool` (`monthRE`),
  `go/internal/podman` (`Event`, `emit`), `go/internal/space` (`on.type`, templates),
  `go/internal/dispatch` (`on.data`), `go/internal/harness/claude.go` (the `result` line),
  `go/internal/supervise` (`run.*` data)

## 1. Where the spec lives

**Recommendation: a new small public repo, `anfs-io/events`**, tagged `v1.0.0` when the person
approves the text. It holds only the contract: `SPEC.md`, `REGISTRY.md` (namespaces and
extensions), `schema/event.v1.json`, `examples/` with `examples/index.json` (**the conformance
fixtures**, 3), `CHANGELOG.md`, a CI workflow that checks the examples against the schema. No
implementation code.

Why: cnfs's gem and lgat's indexer pin a tag and vendor the examples into their own tests; a tag
on a repo that holds nothing else is the version, and its history is the spec's history. It is the
review's §7.4 item 1: the fixtures are promoted out of runtime's `internal/cloudevent/testdata`
(today one file, `bash.jsonl`) into the spec repo, and **both anfsd and the cnfs gem test against
the same tagged copy**. The repo is in anfs-dev's `space.yml` beside docs and runtime, so jobs work
in it as usual; who owns its versions is open question 7.

*Alternatives:* **docs** (`anfs-io/docs`, already public): no new repo, but docs holds plans and
logs, tags there would mean nothing, and consumers would vendor a directory out of a repo that
changes daily. **runtime** (`spec/` beside `go/`): the Go tests read the examples directly, but
the spec's version would be tied to anfsd's releases (v0.3.0 today) and a Rails or Solana consumer
would depend on the daemon's repo. The runtime vendors the examples instead (3).

01's plan keeps its table as history, with a line pointing to the repo; `events.sh`'s and
`cloudevent.go`'s headers point to the repo, not to 01.

## 2. What the spec says

`SPEC.md` is short, normative (MUST/SHOULD, RFC 2119), language-neutral. Its sections:

### 2.1 The event

One CloudEvents 1.0 event (spec 1.0.2) in the structured JSON format. The profile tightens
CloudEvents; it never loosens it. The envelope check stays strict; what opens is who may use it.

| Attribute | Profile | Rule |
| --- | --- | --- |
| `specversion` | required | `"1.0"` |
| `id` | required | UUIDv7 (RFC 9562), lower-case, hyphenated. Globally unique, so `id` alone (not `source`+`id`) is the dedup key. Fresh ids are random; derived ids are allowed (2.9) |
| `source` | required | `/<layer>/spaces/<space id>[/<segment>…]` or `/<layer>/host[/<segment>…]` (2.3). anfs writes exactly `/anfs/spaces/<id>` and `/anfs/host`, as today |
| `type` | required | `io.<layer>.<segment>(.<segment>)+`, segments `[a-z][a-z0-9-]*` (2.2) |
| `subject` | optional | non-empty string, in the type's own terms |
| `time` | **required** (01: "optional, always set") | UTC, RFC 3339, exactly milliseconds and `Z`; its milliseconds equal the id's 48-bit timestamp. When the occurrence happened as well as the producer knows it |
| `datacontenttype` | **required** (01: "always set") | `"application/json"` |
| `data` | **required** (01: `{}` when none) | a JSON object (never an array, a scalar or `null`), top-level keys `^[A-Za-z_][A-Za-z0-9_]*$`, values any JSON; the type's owner defines it (2.2). `io.anfs.*`: flat strings, by convention and by anfs's catalog |
| `dataschema` | not used in v1 | readers ignore it |
| `actor` | extension, required | who acted: `<kind>[:<name>]`, `^[a-z][a-z0-9-]*(:[^\s]+)?$`, at most 128 bytes (`claude:2e9431a0`, `shell:rob`, `podman`, `anfsd`). Self-asserted in development, attested by the publisher in production (2.4) |
| `actorclaimed` | extension, optional | the actor a producer claimed when an attesting publisher set a different `actor` (2.4); same pattern; only when it differs from `actor` |
| `actorspace` | extension, optional | the space id the actor acted as a member of; only with `actorrole` |
| `actorrole` | extension, optional | the role, `^[a-z0-9][a-z0-9._-]*$`; may appear alone (a role in a space with no id, as the bash emitter writes it) |
| `spacepath` | extension, optional | the space's path when written (`anfs-dev`, `cnfs/apps`); informational only |
| `traceparent` | extension, optional | CloudEvents' distributed tracing extension: W3C Trace Context, `^00-[0-9a-f]{32}-[0-9a-f]{16}-[0-9a-f]{2}$`, trace and parent ids not all zeros (2.10) |
| `tracestate` | extension, optional | W3C Trace Context `tracestate`, at most 512 bytes; only with `traceparent` |
| `anfsv` | extension, required | integer `1` |

Global rules:
- **No `null` for any attribute or extension** (CloudEvents allows `null` for unset; we require
  the attribute left out). Inside `data`, `null` is the type owner's choice; anfs's types never use
  it.
- Strings contain no U+0000–U+001F or U+007F–U+009F (CloudEvents' String type) in attributes;
  strings in `data` may contain them escaped (JSON).
- **Numbers in `data`:** integers beyond ±(2^53 − 1) MUST be strings (u64 amounts, lamports):
  JSON readers in JavaScript, jq and Go's `any` lose them otherwise.
- Every top-level member that is not a known attribute is an extension: its name is
  `^[a-z0-9]{1,20}$` and its value a string, integer or boolean (CloudEvents' rules made MUST).
- **At most 4096 bytes** serialized, with the newline, **everywhere** (the spool and the bus, by
  either way in): an event says what changed; detail is read from the item (principle 5). The
  stream enforces it for direct publishes (2.7).
- Readers ignore attributes they do not know and data keys they do not know; they skip, and log
  once, an event whose `anfsv` they do not know.

**"Flat strings, thin events" is anfs's convention, not the envelope's.** Every `io.anfs.*` type in
`REGISTRY.md` has a flat object of string values (the bash emitter writes nothing else, and
dispatch's `on.data` and templates read top-level strings), and the validator holds `io.anfs.*` to
it (rule `anfs.data.flat`). Another layer's data is its own JSON object: cnfs maps a `Rails.event`
payload, lgat an Anchor event, nested where that is natural, within the 4096 bytes. Thin is still
the advice for every layer.

*Alternatives:* flat strings for every layer (01's rule, this plan's first draft): bash and `grep`
win, but cnfs and lgat would flatten and stringify payloads that are naturally structured, and
every consumer would parse them back; decided otherwise (F2). CloudEvents' `authcontext` extension
(`authtype`, `authid`) instead of `actor`: decided otherwise (actor and role separate, AGENT-NATIVE
"Decided"); a bridge may add `authtype`/`authid` beside ours.

### 2.2 Types: names and ownership per layer

- `io.<layer>.…`, each layer (namespace) owned by one layer and listed in `REGISTRY.md`: `anfs`
  (anfs-io), `cnfs`, `lgat`; `rws` and `roteoh` reserved for theirs. A tool that is not a layer
  (Jev, a tracker) registers a namespace the same way when it publishes types of its own. Only the
  owner defines types in its namespace and documents them; anfs's catalog (every `io.anfs.*` type
  with its subject and data keys: `report.*`, `source.updated`, `run.*`, `container.*`, `pod.*`,
  `podman.*`) is in `REGISTRY.md`; layers keep theirs in their own repos and link them.
- Inside a namespace the owner chooses the depth: anfs uses `<noun>.<verb>`; cnfs
  `<engine>.<noun>.<verb>` (`io.cnfs.partitia.report.filed`, `io.cnfs.chatter.message.posted`);
  lgat `<program>.<event>` (`io.lgat.escrow.payment.settled`, mapped from Anchor's
  `PaymentSettled` by a table, not mechanically). Verbs are past tense: an event is a fact.
- **The short type** drops `io.anfs.` for anfs's types and `io.` for every other namespace:
  `report.filed`, `cnfs.partitia.report.filed`. It is what `--type` filters, dispatch rules'
  `on.type` and NATS subjects (2.5) use. To keep it unambiguous, **no `io.anfs` type's first
  segment may be a registered namespace** (no `io.anfs.cnfs.*`).
- A type's data may grow (new keys); removing a key or changing its meaning needs a new type.
- **Run usage (review §5.3).** anfs's catalog gives `run.completed`, and `run.failed` when known,
  these data keys (strings, as every `io.anfs` value), each left out when the harness does not
  report it: `model` (the model the harness reports it used, else the rule's), `input_tokens`
  (every prompt token the model processed, cached or not), `output_tokens`, `cost_usd` (decimal,
  the harness's own figure, an estimate). They map one to one onto OTel's generative-AI
  conventions (`gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`)
  for a later bridge (§9.1); anfsd fills them in 05b.

*Alternatives:* one namespace for everything (`io.anfs.cnfs.…`): anfs would review every layer's
types; prefixing by domain we own (`io.anfs`, `io.cnfs` read as reverse DNS of anfs.io, cnfs.io):
CloudEvents only says SHOULD; open question 2.

### 2.3 Space ids and sources

**The space-id rule** (documented, one rule for every layer):
- A space id is 10 characters of `[a-z0-9]` (`^[a-z0-9]{10}$`); readers accept that. anfs
  generates them from `a-z2-7` (32^10 ≈ 2^50, random: `anfs_space_id_new`, `wsm id`); another
  generator MUST be random over at least that alphabet, so ids stay unique across layers without
  coordination.
- Made once, **never changed and never reused**, kept by renames and moves; `host` is reserved.
- It is the scope everywhere: the spool directory (2.6), the second token of a NATS subject (2.5),
  `actorspace`, `accepts:` later. A layer's tenant that is a space has one (§7.3: "tenant =
  space"), stored beside the tenant's own primary key; open question 8 has the reasoning.

**Sources, per layer:** `/<layer>/spaces/<space id>` or `/<layer>/host`, optionally followed by
`/<segment>`s (`[A-Za-z0-9._~-]+`) the layer owns:
`^/[a-z][a-z0-9-]*/(host|spaces/[a-z0-9]{10})(/[A-Za-z0-9._~-]+)*$`.
- `<layer>` names the producing layer (`/cnfs/spaces/k3x7q2mbzp/partitia`: cnfs's partitia app in
  that space). It SHOULD be the type's namespace; it differs when one layer's producer emits
  another's type (anfs's `anfs report`, run in a cnfs space, still writes `/anfs/spaces/<id>`).
  Validators check the grammar, not the registry, so a new layer needs no validator release.
- **The scope** of an event is the space id in its source, or `host`, whatever the layer: where an
  event happened is a space, which is an ANFS concept shared by every layer.
- anfs writes exactly `/anfs/spaces/<id>` and `/anfs/host`: every line written so far is unchanged.

*Alternatives:* one source form for every layer (`/anfs/spaces/<id>`, this plan's first draft):
simpler, but the source no longer says who produced the event, which CloudEvents' `source` is
for, and a cnfs app has nowhere to name itself; decided otherwise (F2). A free-form source with the
scope in an extension (`space`): CloudEvents-idiomatic, but a second place for the same fact and
every reader changes.

### 2.4 Actors: asserted in development, attested in production

- `actor` (and `actorspace`, `actorrole`) say who acted. **On a dev host they are self-asserted**:
  any spool writer or bus publisher sets them, and nothing checks. That is acceptable only on a
  single person's development host (F6: no signing is built), and the spec says so.
- **In production the publisher attests the actor.** The component that puts an event on a
  production bus (cnfs's outbox relay, lgat's indexer, a bridge) sets `actor`, `actorspace` and
  `actorrole` from its own authenticated context (the session or job that made the outbox row, the
  indexer itself), never from a caller's payload or claim; the bus authenticates the publisher (a
  NATS user per publisher, 2.7). Consumers in production trust `actor` only as far as they trust
  its publisher. **cnfs and lgat MUST NOT design around self-asserted identity.**
- **`actorclaimed`:** an attesting publisher that receives a different claim keeps it there, so
  the record shows the attempt instead of erasing it. First user: 02d, where anfsd stamps a run's
  events with the run's real actor and keeps a forged claim in `actorclaimed`. Readers show
  `actor`; `actorclaimed` is evidence (a view may flag it).
- On the dev bus, a layer's publisher user (2.7) attests the **layer** (it can publish only its
  layer's subjects), not the actor.
- When 04's approvals arrive, they come through `anfs inbox`, never from a spool line's actor
  (the review's F6).

### 2.5 Subjects on NATS

- **The binding:** CloudEvents' NATS protocol binding, **structured mode only**: the body is the
  event (the spool line as it is), header `Content-Type: application/cloudevents+json`. Binary
  mode (`ce-*` headers) is not published; a consumer that receives one may refuse it. The
  binding says nothing about subjects; the subject scheme is ours.
- **Header `Nats-Msg-Id: <id>`** on every publish (JetStream's dedup).
- **Subject: `anfs.<scope>.<short type>`** (02's, unchanged): `anfs.tkj76y3nfx.report.filed`,
  `anfs.host.container.started`, `anfs.k3x7q2mbzp.cnfs.partitia.report.filed`. This is review
  §11 q1's first option (`anfs.<scope>.<layer>.<noun>.<verb>` in one stream), with anfs's own
  `anfs` token left out as the short type already does. `anfs.` names the bus and the protocol
  (which every layer uses), not the type's owner. Filters: `anfs.<id>.>` a space,
  `anfs.*.cnfs.>` a layer, `anfs.*.report.>` an anfs kind. **One stream** (`ANFS`, `anfs.>`)
  holds every layer.
- **Why one stream and this mapping** (open question 6):
  - JetStream's dedup by `Nats-Msg-Id` is per stream: one stream is what makes "dedup by id"
    hold across layers and across both ways in (2.7) without a second mechanism.
  - The space is the unit of grants (AGENT-NATIVE, the layering rule): a space's permission
    (`accepts:`) or filter stays one subject, `anfs.<id>.>`, covering every layer's events in it;
    a per-layer root would make it `*.<id>.>`, which also matches any other root.
  - A layer's publish permission is still one subject, `anfs.*.<layer>.>` (2.7): the namespace
    rule of 2.2 keeps it from matching anfs's types.
  - Nothing that exists changes: anfs's subjects, the ui's and dispatch's filters, the recorder's
    one consumer, one set of limits and retention.
  - In production a layer runs its own NATS (or none); the same scheme carries over as is.
- *Alternatives:* a root and stream per layer (`cnfs.<scope>.<rest>`, stream `CNFS`): symmetric
  and separately sized, but dedup splits per stream, a space's filter becomes `*.<id>.>`, the
  recorder and the ui follow several streams, and streams cannot overlap subjects, so nothing can
  later span layers in one; `anfs.<scope>.<type without io.>` (`anfs.<id>.anfs.report.filed`):
  uniform, but every existing subject, the ui's filter and dispatch's filters change.

### 2.6 The first way in: the spool (any language, no client)

On a host with anfs, a producer appends to the spool and anfsd ships it; it needs no NATS client.

- **Path:** `${XDG_STATE_HOME:-$HOME/.local/state}/anfs/events/<scope>/<YYYY-MM>.jsonl`; `<scope>`
  the source's space id or `host`; the month the event's `time` (UTC). The directory is created
  when missing (0755), the file 0644. `<YYYY-MM>.bus.jsonl` beside it is the recorder's (2.7):
  producers never write it.
- **Finding the scope:** the `id:` (top level of `space.yml`, 2.3) of the nearest `space.yml` at
  or above the producer's directory (a Rails app's root); none, or no id: the host scope.
  `spacepath` is optional for other producers.
- **One event per line**, UTF-8, `\n`-terminated, in the **canonical form**: compact JSON (no
  insignificant whitespace), `data` the last member, `"anfsv":1` as written. The bash reader
  relies on it (`_anfs_ev_parse` reads the attributes before `,"data":`, whatever data holds);
  Ruby's `JSON.generate`, Python's `json.dumps(…, separators=(",", ":"))` and Go's encoder produce
  it with the hash built in order.
- **Atomic append, no locks:** the whole line in **one `write(2)`** on a descriptor opened with
  `O_APPEND`, at most 4096 bytes; a local filesystem (not NFS). Per language: bash `cat >> f <<<
  "$line"` (not `printf`: macOS splits it), Go `os.OpenFile(…O_APPEND…).Write`, Ruby
  `File.open(f, File::WRONLY|File::APPEND|File::CREAT, 0o644) { _1.syswrite(line) }`, Python
  `os.write(fd, line)`.
- **Never** rewrite, truncate or delete a line; files are kept (decided in 02). An emit never fails
  its caller: a refused or failed write is a warning.
- Non-canonical lines are still CloudEvents, and anfsd ships any valid event; only the bash reader
  needs the canonical form, so `anfs events validate` reports it (as `noncanonical`).

### 2.7 The second way in: publishing to NATS, and the recorder

Direct publishes are **first class**, not a workaround: in production a cnfs app publishes to NATS
(or its outbox relay does), so on the dev host the gem must be able to build against that
interface too (F3). Both ways are equal; a producer uses one per event (both for the same id is
harmless, deduped, and pointless).

**The publish contract** (the same on the dev bus and a production one):
- Subject `anfs.<scope>.<short type>` (2.5), agreeing with the event's `source` and `type`.
- Body: the event, valid, at most 4096 bytes; SHOULD be the canonical form (2.6).
- Headers `Content-Type: application/cloudevents+json` and `Nats-Msg-Id: <id>`.
- **A JetStream publish, awaiting its ack**, retried with the same body until acked: a core NATS
  publish with no ack is at most once and does not meet 2.8. The outbox row is marked sent only
  after the ack.

**The publisher user, per layer (dev bus):** core/nats makes one NATS user per layer that
publishes directly, named after its namespace (`cnfs`, `lgat`), its password in
`~/.local/state/nats/users/<layer>` (0600, like the others), from a list of layers in nats's
settings. Its permissions: **publish `anfs.*.<layer>.>`** only (never `io.anfs.*` or another
layer's subjects), **subscribe `anfs.>`** and `_INBOX.>`, and the JetStream API a consumer needs on
`ANFS` (as `reader` has: its own durable consumers, fetch, ack); never stream management. It is
distinct from `anfsd` (ships the spool, manages the stream, publishes everything), `admin` (the
person's) and `reader` (subscribe only: the ui). *Alternative:* one shared `tools` user: one
password, but any tool could publish any layer's types, and the bus could no longer attest the
layer (2.4).

**The recorder** keeps "the files are the complete record" true for both ways in:
- A durable consumer of anfsd's, `anfsd-recorder`, on `ANFS`, filter `anfs.>`, deliver all when
  first made, explicit ack.
- For each message: parse it against the profile and check its subject against it. Invalid or
  mismatched: log once (rule and subject), terminate the message (it stays in the stream, unknown
  to the record). Its id already in its month's files (`<scope>/<YYYY-MM>.jsonl` or
  `<YYYY-MM>.bus.jsonl`, the scope and month from the event): ack, nothing written; **this is how
  everything the shipper published from the spool is skipped**, with no header to trust or forge.
  Otherwise: append its canonical form (the same members and values; whitespace removed and
  `data` moved last when the publisher did not) to **`<scope>/<YYYY-MM>.bus.jsonl`** in one
  `O_APPEND` write, then ack.
- Recorded events go to their own file so **the shipper never re-publishes them** (it reads only
  `<YYYY-MM>.jsonl`, `spool.monthRE`): past the 24 h dedup window a re-publish would be a
  duplicate on the bus.
- **Readers of the record read both files** of a month; their lines are in append order, so a
  reader that wants time order sorts by `id`. `anfs events ls` and `-f` do (05b).
- The record is complete within the stream's retention (`ANFSD_STREAM_MAX_AGE`, 90 days): a
  recorder down longer loses what the stream dropped meanwhile. A dev host only: in production the
  record is the layer's own store.
- *Alternatives:* the recorder appends to the spool file itself, the shipper skipping recorded
  lines by an index of ids (one file per month, but the shipper grows a second state and a crash
  between them re-publishes); a header marking spool-shipped messages (`Anfs-Origin: spool`) to
  skip them (cheaper, but a publisher can set it and drop its events from the record).

### 2.8 Delivery semantics

- **At least once**, end to end, by either way in. Dedup by `id`: JetStream drops a repeat inside
  its window (24 h on the dev host); the recorder by its files; every consumer is idempotent by
  `id` beyond it (records what it has handled).
- **Distinct occurrences, distinct ids.** The same fact re-sent is the same id; a new fact about
  the same thing (lgat's `finalized` after `confirmed`) is a new event with a new id, correlated
  by `subject` and data.
- **Ordering:** per scope in append order on one host for the spool; none across scopes or
  between the two ways in; `id` order is time order to the millisecond. A consumer that needs
  causal order uses data (`slot`, a sequence).
- Consumers ack after handling. Append-only: no event is retracted; a correction is a new event.

### 2.9 Derived ids (re-derived events)

A producer that re-derives events from an external log (the indexer re-reading the chain after a
crash, an outbox replay) must produce the **same id** for the same occurrence, or dedup fails. Two
ways, both conformant: persist the id with the record (an outbox row), or **derive it**: a UUIDv7
whose 48-bit timestamp is the occurrence's time (`time`'s milliseconds) and whose 74 other bits
come from SHA-256 of a stable key (`io.lgat.escrow.payment.settled|<signature>|<ix>|<event
index>|finalized`). It departs from RFC 9562's SHOULD that those bits be random, knowingly (open
question 4). *Alternative:* UUIDv5/v8 for derived events: breaks "id is a UUIDv7, ordered by time".

### 2.10 Extensions: ours, tracing, and the layers'

- anfs's eight (2.1: `actor`, `actorclaimed`, `actorspace`, `actorrole`, `spacepath`,
  `traceparent`, `tracestate`, `anfsv`) are part of v1. A layer registers its own in
  `REGISTRY.md` (name, owner, type, values, which namespaces carry it) by a PR to the spec repo;
  adding one is a minor version of the spec (`v1.1.0`), never an `anfsv` bump, because v1 readers
  ignore unknown extensions.
- Names are unprefixed but registered, and must not take a name CloudEvents' documented extensions
  define (`authtype`, `sequence`, `dataref`, …: checked against their list when registering).
- **Tracing, reserved now (review §5.3, §9.1):** `traceparent` and `tracestate` are CloudEvents'
  distributed tracing extension, with its meaning: the trace context of the code that produced
  the event, so a consumer's work joins the producer's trace. They travel in the event (the
  structured body), which is what carries them across the spool, the outbox and a job boundary;
  a publisher MAY also copy `traceparent` into a NATS header for its own instrumentation, and
  consumers read the attribute. anfs's emitters do not set them yet; cnfs's gem sets them from the
  `Rails.event` context; anfsd may later pass a triggering event's `traceparent` to the run it
  starts and onto its `run.*`.
- **The worked example, lgat's `commitment`:** String, `confirmed` or `finalized` (Solana's levels;
  `processed` is not published), on `io.lgat.*` only. Money moves only on `finalized`; a
  `confirmed` that never becomes `finalized` simply has no successor.
- *Alternative:* prefixed names (`lgatcommitment`): no registry needed, collision-free, but long
  (CloudEvents' SHOULD of 20 characters) and ugly in every filter.

### 2.11 Versioning and compatibility

- `anfsv` is the envelope's version (1). The spec's tags are semver: `v1.y.z` is `anfsv` 1;
  `v2.0.0` would be `anfsv` 2. Who cuts them: open question 7.
- **No bump** for what every v1 reader already tolerates: a new type, a new data key, a registered
  extension, a new namespace (validators check grammar, not the registry). **A bump** for anything
  that makes a v1-valid event invalid or changes an attribute's meaning (`data` other than an
  object, a new source form, a larger size limit).
- **Deploy order on a bump:** readers and shippers first, writers last (anfsd skips and logs an
  `anfsv` it does not know, and its cursor moves on). A writer writes the lowest `anfsv` that
  expresses the event.
- The 01 → 05 changes are **not** a bump: the tightening (`time`, `datacontenttype`, `data`
  required; no `null`; actor pattern) holds for every line any implementation has written (the
  35 lines on this host, re-checked with jq on 2026-10-24, all do), and the opening (other
  namespaces and sources, JSON data, new extensions) only accepts more, before v1.0.0 is tagged.

## 3. Conformance

The fixtures live in the spec repo and are the one source of truth; every implementation vendors
a tag's copy with a `VERSION` file (runtime `internal/cloudevent/testdata/spec/`, core
`tests/fixtures/events-spec/`, the cnfs gem's `spec/fixtures/`), and updating is copying a newer
tag's.

- **Examples** (`examples/valid/*.jsonl`, `examples/invalid/*.jsonl`, one line each, so line-level
  rules are testable). Valid: `report-filed` and `report-filed-test` (the two lines of runtime's
  `bash.jsonl`, promoted), `report-closed`, `source-updated`, `run-completed` (actor `anfsd`, with
  the usage keys), `container-died` (host scope, podman), `as-role` (actorspace + actorrole),
  `role-only`, `no-subject`, `unknown-extension`, `traceparent` (with `tracestate`),
  `actorclaimed`, `cnfs-partitia-report-filed` (source `/cnfs/spaces/<id>/partitia`, nested data),
  `cnfs-chatter-message-posted`, `cnfs-host` (`/cnfs/host`), `lgat-payment-confirmed` and
  `-finalized` (derived ids, `commitment`, an amount as a string), `data-null-value` (a cnfs type),
  `max-size` (exactly 4096 bytes), `noncanonical` (valid event, whitespace: valid, flagged for the
  spool). Invalid, one rule each: missing `specversion`/`id`/`source`/`type`/`time`/`actor`/
  `anfsv`/`data`; `anfsv` 2; `specversion` "0.3"; UUIDv4 id; upper-case id; source `/x`, `/anfs/
  spaces/short`, `/Cnfs/host`; type without `io.`; type with an upper-case segment; time without
  ms, with an offset, not matching the id; `null` subject; empty subject; `data` an array, a
  string, `null`; a bad top-level data key; an integer over 2^53 in data; `io.anfs` with nested
  data and with a numeric value (`anfs.data.flat`); bad extension name (`actor_role`);
  object-valued extension; bad actor; bad `actorclaimed`; actorspace without actorrole; bad
  `traceparent`; `tracestate` without `traceparent`; 4097 bytes; two lines.
- **`examples/index.json`**: per file `{file, valid, rule, schema, subject}`: `rule` a stable code
  (`time.ms`, `size.max`, `anfs.data.flat`), `schema` whether the JSON Schema alone catches it
  (size, one line, time = id's milliseconds, big integers and canonical form it cannot), `subject`
  the NATS subject of a valid example (2.5), so the subject mapping is conformance-tested too.
- **`schema/event.v1.json`**: JSON Schema draft 2020-12, `$id` the raw GitHub URL of the tag (open
  question 1 has the name): the table of 2.1, `patternProperties` `^[a-z0-9]{1,20}$` with scalar
  values for extensions and `additionalProperties: false`, the UUIDv7 pattern
  `^[0-9a-f]{8}-[0-9a-f]{4}-7[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$`, the source pattern of
  2.3, `data` an object with `propertyNames` and, under an `if` on `^io\.anfs\.`, string values
  only, the trace patterns, `commitment`'s enum. The spec repo's CI runs a pinned
  `check-jsonschema` over the examples (valid pass; invalid with `schema: true` fail). Any language
  validates with its own library (Ruby `json_schemer`).
- **The runnable check: both.** `anfsd validate [FILE...|-]` (runtime, `internal/cloudevent`, the
  full profile, line-level rules included, and with a spool path the scope directory and month
  file, `.bus.jsonl` included) prints `FILE:LINE RULE message`, exit 1 on any invalid; **`anfs
  events validate [FILE...|SPEC...|--all]`** (core) is the person's and agents' form: it resolves
  spool files as `events ls` does and runs `anfsd validate` (with core/anfsd absent: an error
  naming it; bash does not re-implement the validator). **Go tests** in runtime run every example
  through `Parse` and, test-only, through the schema (`santhosh-tekuri/jsonschema`), asserting the
  manifest (and `NatsSubject()` against `subject`): the hand-written parser and the schema cannot
  drift. Core's bats run `_anfs_ev_parse` over every valid example (other layers' sources and
  nested data read right), `anfs events validate` with a stub `anfsd` (argument passing, exit
  codes), and the bash emitter's output through a fixture check of the canonical form. **The cnfs
  gem's RSpec** runs its mapper's output and every example through `json_schemer` plus the line
  rules, asserting the same manifest.
  *Alternatives:* Go tests only (no tool for cnfs's developer to check their gem's spool lines on
  a dev host); a bash validator (a second, weaker implementation of the rules).

## 4. Making the implementations conform: the gap list (05b)

Re-checked 2026-10-24 by reading the code (read-only) at the same commits, against this revision;
the real spool's 35 lines all conform.

**runtime `internal/cloudevent` and its users**
1. `typeRE` accepts only `io.anfs.*`: another layer's spool line is logged as not an event and
   never shipped. Accept `io.<layer>.…` (2.2); `Short()` strips `io.anfs.` or `io.`, so
   `NatsSubject()` follows 2.5 (today an `io.cnfs` type would give `anfs.<id>.io.cnfs.…`).
2. `source` must be `/anfs/spaces/<id>` or `/anfs/host`: accept 2.3's per-layer form, the scope
   from it.
3. `data` must be a flat object of strings, held as `map[string]string`: accept any JSON object
   (2.1: top-level key rule, no integer over 2^53), keep it raw (`json.RawMessage`) beside a map
   of its top-level string values; hold `io.anfs.*` to flat strings (`anfs.data.flat`). Dispatch's
   `on.data` and `{{data.<key>}}` (`internal/dispatch`, `internal/space/template.go`) read top-level
   string values only: a non-string value never matches and renders empty, documented.
4. `Parse` accepts `null` for string attributes and data values (`json.Unmarshal` of `null` into a
   string is no error) and an empty `subject`: refuse (data's own `null`s are the owner's, 2.1).
5. `time` optional with any fraction: required, exactly milliseconds, equal to the id's timestamp.
6. `data` and `datacontenttype` may be absent: required.
7. `actor`, `actorspace`, `actorrole`, `spacepath` are validated by `Line` but not by `Parse`;
   `actorspace` without `actorrole` is not refused; `actorclaimed`, `traceparent`, `tracestate`
   (2.1) are not known.
8. Unknown members are not checked: extension names (`^[a-z0-9]{1,20}$`) and scalar values.
9. CloudEvents' forbidden control characters (U+007F–U+009F) in string attributes pass `Line` and
   `Parse`.
10. `anfsd validate` does not exist; `testdata/bash.jsonl` is replaced by the vendored examples
   (`testdata/spec/`, 3).
11. The shipper does not check that a line's scope directory matches its `source`: log once, ship
   by `source`.
12. `internal/space` trims only `io.anfs.` from `on.type` (`space.go`): rules on other namespaces
   use the short type of 2.2 (works by accident today through `Short()`; made explicit and
   tested).
13. **The recorder** (2.7) does not exist: `anfsd-recorder`, the id check over a month's two files
   (an index per month file, built lazily and followed by offset), `<YYYY-MM>.bus.jsonl`, the
   canonical re-encoding, terminate on invalid or mismatched; `anfs bus status` shows its row.
   `spool.monthRE` already excludes `.bus.jsonl` (the shipper never reads it): kept, and tested.
14. `bus.EnsureStream` sets no `MaxMsgSize`: set it so the stream refuses a direct publish over
   4096 bytes (to confirm in the job whether NATS counts headers in it; if so, 4096 plus the two
   headers' room, and the recorder checks the body).
15. Run usage (2.2) is not recorded: `harness` extracts it where the harness reports it (claude's
   `result` line has `usage` and `total_cost_usd`, today only rendered into the summary; the
   others where their output gives it), `supervise` puts `model`, `input_tokens`,
   `output_tokens`, `cost_usd` on `run.completed` and `run.failed`.
16. Optional: podman events carry podman's own `timeNano`; `time` is when anfsd wrote the line
   (milliseconds later). Use podman's time when present (2.1: "when the occurrence happened").

**core**
17. `events.sh`: `ANFS_ACTOR` is taken verbatim: check the actor pattern, warn and fall back to
   detection.
18. `events.sh`: `_anfs_ev_str` escapes U+0001–U+001F but passes U+007F and C1 controls, and bytes
   that are not UTF-8 (an invalid JSON text): refuse with the usual warning (a check under a UTF-8
   locale, no fork; to confirm in the job).
19. `events.sh`: the reader's short type strips only `io.anfs.`, and `EV_SID` is read only from
   `/anfs/spaces/…`: strip `io.` for other namespaces (2.2) and take the scope from any layer's
   source (2.3), so `--type 'cnfs.*'` and the SPACE/EVENT rows read right.
20. `events.sh`: `events ls` and `-f` glob `*.jsonl` per scope, so they would read
   `<YYYY-MM>.bus.jsonl` already, but a month's two files come out one after the other: merge them
   in `id` order, follow both, and keep `--since`'s month comparison right for the `.bus` name.
21. `bus.sh`: the layers' publisher users (2.7): a list of layers in nats's settings
   (`ANFS_BUS_LAYERS`, empty by default), a password file each, their permissions in the rendered
   `server.conf`; `anfs bus status` names them.
22. `anfs events validate` (new, 3); headers of `events.sh` and `cloudevent.go`, AGENTS.md
   "Events" and "The bus" (the two ways in, the recorder, the users), nats's README and 01's plan
   point to the spec repo; a comment says "recorded as null" where the space is in fact left out.

**Not in 05b:** stamping a run's actor and keeping `actorclaimed` (02d); `traceparent` from anfs's
emitters (later, 2.10).

**Not gaps:** the envelope's order and compact form, UUIDv7 generation, the 4096 limit in the
spool, one `O_APPEND` write, `Nats-Msg-Id`, the `Content-Type` header, the podman mapping's types,
subjects, actor and string data, `run.*` from the supervisor, `_anfs_ev_parse` cutting at the first
`,"data":` (data is last whatever it holds).

## 5. What cnfs and lgat need

**cnfs, the events gem (S9; review §7.2): it starts as soon as 05a is tagged, in parallel with 02d
and 05b.** Its four stages are `Rails.event` → map (an allow-list to `io.cnfs.<engine>.<noun>.<verb>`)
→ outbox (a row in the same transaction, id persisted, 2.9) → transport. Emit, map, outbox, the
inbound side (events → ActiveJob, idempotent by id in an inbox table, 2.8) and the actor and
tenant context are tested against the tagged fixtures alone: they need neither anfsd nor NATS.
**Only the dev transport needs anfsd changes (05b):**
- spool append (2.6): needs gaps 1–3 so anfsd ships `io.cnfs` lines with cnfs sources and JSON
  data;
- NATS publish (2.7): needs the `cnfs` publisher user (21) and the recorder (13) on the dev host;
  until then the gem's own tests run against a throwaway nats-server with a stream `ANFS` on
  `anfs.>` and the contract of 2.7, as the ui's specs do.

What it gets from 05a:
- `anfs-io/events` at `v1.0.0`: SPEC.md, the schema, the examples and the manifest to vendor
  into RSpec (`json_schemer` plus the line rules); `io.cnfs` registered.
- Both ways in, each a transport: the spool with Ruby's append and the scope rule (the app's
  nearest `space.yml` id), and the publish contract (subject, structured mode, `Nats-Msg-Id`,
  JetStream ack) which is also production's. A Postgres store is theirs: only the envelope and the
  dedup-by-id semantics carry over.
- Its data is its own JSON object (2.1), thin by advice; `traceparent` from the `Rails.event`
  context (2.10).
- **The actor is attested by its publisher in production** (2.4): the outbox row records the
  authenticated actor and role when the change is made, and the relay publishes those, never a
  caller's claim; in development the gem's actor is self-asserted.
- `anfs events validate` on their dev host to check what the gem writes.
- Not needed: anfsd's code, the anfs tool suite in production, NATS in development (the spool).

**lgat's indexer:**
- `io.lgat` and the `commitment` extension registered; the type mapping from Anchor events is
  theirs (a table).
- Derived ids (2.9) so a re-index dedups; `time` = block time (seconds, `.000`), `subject` = the
  transaction signature, data a JSON object (amounts as decimal strings, being u64; keys base58;
  `slot`, `ix` as integers); `confirmed` and `finalized` as two events (2.8); actor names the
  producer (`lgat-indexer`), which as the publisher attests it (2.4), the signer in data. Large
  Anchor payloads reduced under 4096 bytes. In development it publishes as the `lgat` user or
  appends to the spool.

## 6. Slicing, tests, by hand, open questions

**Two slices, each its own job:**
- **05a — the spec repo** (job `agents-05a events docs`): SPEC.md (2 in full: the opened envelope,
  space ids and sources, actors and attestation, subjects, both ways in with the recorder and the
  publisher user, delivery, derived ids, extensions with tracing, versioning), REGISTRY.md
  (namespaces, extensions, anfs's type catalog from `ANFS_EVENT_TYPES` with `run.*`'s usage
  keys), the schema, the examples (runtime's `bash.jsonl` promoted), the manifest, CI with a pinned
  `check-jsonschema`, README; 01's pointer. Not tagged until the person has read SPEC.md.
- **05b — conformance** (job `agents-05b runtime core docs`, after 05a's tag): gaps 1–22,
  `anfsd validate`, the recorder, the publisher users, the vendored examples in runtime and core,
  `anfs events validate`, docs. anfsd 0.4.0 and its pins. If the job runs long, the recorder and
  the users (13, 14, 20, 21) may land as a second commit series in the same job, never as a
  separate plan.

**Tests:** spec CI: every valid example passes the schema, every `schema: true` invalid one fails,
every file in `examples/` is in the manifest, every valid one has a `subject`. runtime: each
example through `Parse` and the schema agrees with the manifest, and `NatsSubject()` with its
`subject`; `Line` refuses each gap's case; the shipper ships an `io.cnfs` line with nested data on
`anfs.<id>.cnfs.…` and logs a scope mismatch once; the recorder (embedded NATS): a direct publish
lands once in `.bus.jsonl` in canonical form, a spool-shipped event is not recorded, a redelivery
after a crash between append and ack writes nothing, an invalid or mismatched message is
terminated and logged once, the shipper never reads `.bus.jsonl`; the stream refuses a 4097-byte
body; a publisher-user config (from core's rendered conf, as a fixture) may publish
`anfs.x.cnfs.y` and not `anfs.x.report.filed`; usage keys from claude's recorded `result` line;
`anfsd validate` output and exit codes; idle CPU unchanged. core: the emitter refuses a bad
`ANFS_ACTOR`, DEL and non-UTF-8 input with a warning and exit 0; every emitted line passes a
stub-free canonical-form check; `_anfs_ev_parse` over every valid example; `events ls --type
'cnfs.*'` and a month with both files in id order; `bus.sh` renders the layer users and their
permissions (nats.bats: `cnfs` publishes its subjects and is refused anfs's); `anfs events
validate` with a stub `anfsd`. While working, through `anfs work run`; core's `tests/all` once at
the end.

**By hand, for the person:**
1. Before 05a: create `anfs-io/events` (public, empty) and add it to anfs-dev's `space.yml`
   `repositories:`.
2. After reading SPEC.md: merge 05a, push, `git tag v1.0.0 && git push origin v1.0.0`.
3. After 05b: runtime push and tag, the sums into `packages/anfsd/pins.yml`, `ANFS_BUS_LAYERS=cnfs`
   in nats's local settings when cnfs is ready to publish, `anfs src update core && ppm install
   core/anfs core/nats core/anfsd`; `anfs events validate --all` is clean and `anfs bus status`
   shows the recorder.
4. Tell cnfs the tag, and that the gem can start on the fixtures now (an inbox message, or by hand
   until 04).
5. The real spool holds a test line from 02a's fixture run (scope `abcde23456`, actor `shell:t`,
   2026-10-23 17:46): valid, but not a real space. No tool deletes events; removing the directory
   is the person's call.

**Open questions for the person** (each with the recommendation):
1. The repo: `anfs-io/events` (recommended) or `anfs-io/spec` (room for the inbox and work-item
   specs later)? **Public from the start** (recommended: cnfs and lgat vendor it, and nothing in it
   is private).
2. Namespaces `io.anfs`, `io.cnfs`, `io.lgat` read as reverse DNS: **keep them** whether or not we
   own those domains (recommended: CloudEvents only says SHOULD, `REGISTRY.md` is what makes them
   unique, and renaming later is a new type for every event).
3. ~~`data` flat strings or any JSON?~~ **Answered (F2):** `data` is any JSON object within 4096
   bytes; flat strings stay anfs's convention for `io.anfs.*`, held by its catalog (2.1).
4. Derived UUIDv7 ids (hash bits instead of random) for re-derived events: **accept**
   (recommended: they keep "id is a UUIDv7 ordered by time" and make re-indexing idempotent; the
   RFC's randomness is a SHOULD, and 74 hash bits do not collide in practice).
5. The 4096-byte limit on the bus too, in production: **yes** (recommended), now enforced on the
   dev stream for direct publishes (gap 14); a larger payload is a reference in data.
6. (Review §11 q1) Subjects for other layers: **one stream, `anfs.<scope>.<short type>`**, i.e.
   `anfs.<scope>.<layer>.<noun>.<verb>` with anfs's own token left out (recommended, 2.5: dedup by
   id holds across layers only within one stream; a space stays one filter and one grant,
   `anfs.<id>.>`; a layer's publish permission is one subject; nothing existing changes), or a
   root and stream per layer.
7. (Review §11 q2) Who owns the spec's versioning: **anfs-io, as the anfs-dev space's** (the
   maintainer role drafts, the person approves and tags, as with any anfs release); layers own
   their namespaces' types and versions in their own repos without a spec release, and change
   the spec itself (a namespace, an extension) by a PR to `REGISTRY.md`, a minor version
   (recommended: the envelope is ANFS's, every layer consumes it, and a committee is premature
   with one person). Revisit if a layer outside the person's spaces adopts it.
8. (Review §11 q4) Space id vs production tenant id: **converge on the rule, not the key**
   (recommended): every id in `source`, `actorspace` and subjects follows 2.3 (10 characters,
   random, never changed), so a production tenant that is a space gets one, generated once and
   stored beside its own primary key, which stays the app's (a UUID, an integer). Mapping the
   app's key into events instead would leak integers that collide across apps into subjects, or
   UUIDs that make every subject and filter 36 characters, and two id grammars into every reader.

## Sources checked (2026-10-23, read-only; re-checked 2026-10-24)

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
- Documented extensions (names to avoid; `authcontext`: `authtype`, `authid`, `authclaims`;
  distributed tracing: `traceparent`, `tracestate` per W3C Trace Context):
  https://github.com/cloudevents/spec/tree/main/cloudevents/extensions,
  https://github.com/cloudevents/spec/blob/main/cloudevents/extensions/authcontext.md,
  https://github.com/cloudevents/spec/blob/main/cloudevents/extensions/distributed-tracing.md,
  https://www.w3.org/TR/trace-context/
- OTel generative-AI semantic conventions (`gen_ai.response.model`, `gen_ai.usage.input_tokens`,
  `gen_ai.usage.output_tokens`): https://opentelemetry.io/docs/specs/semconv/gen-ai/
- `anfs-io/docs` and `anfs-io/runtime` public, `anfs-io/events` and `anfs-io/spec` free:
  https://api.github.com/repos/anfs-io/{docs,runtime,events,spec}
- Code re-read 2026-10-24: runtime `ae71386` (`cloudevent.typeRE`, `Parse`'s source switch and
  `map[string]string` data, `bus.EnsureStream` without `MaxMsgSize`, `spool.monthRE`,
  `space.go`'s `TrimPrefix`, `dispatch`'s `on.data`, `harness/claude.go`'s `result` line,
  `supervise`'s `run.*` data); core `44e456b` (`events.sh`'s `_anfs_ev_parse` and `*.jsonl` globs,
  `bus.sh`'s users `anfsd`, `admin`, `reader`).
