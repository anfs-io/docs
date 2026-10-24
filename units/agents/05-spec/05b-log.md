---
plan: agents/05-spec (05b)
job: agents-05b
date: 2026-10-24
---

# Log: agents/05b, the implementations conform to the spec

## What was built

The spec is anfs-io/events as 05a built it. It is **not tagged yet**: both implementations vendor commit `90371240fd822b3f1e4ae693a42aead9568564a3` (branch `agents-05a`). Each copy has a `VERSION` file that names that commit. After the tag, copy the tagged `schema/` and `examples/` over both copies, with a new `VERSION`.

**anfs-io/runtime** (anfsd):

- **`internal/cloudevent` follows the spec** (gaps 1–9, 12, 23):
  - Types are any `io.<namespace>`, and sources are per layer (the scope is taken from either form).
  - `data` is any JSON object. `Event.RawData` keeps it as written; `Event.Data` holds its top-level strings. `io.anfs.*` must be flat strings.
  - `Parse` refuses:
    - any `null` attribute, and an empty subject;
    - a `time` that is missing, not exactly milliseconds, or not equal to the id's milliseconds;
    - a missing `datacontenttype` or `data`;
    - any of the profile's extensions that does not match its pattern;
    - `actorspace` without `actorrole`, and `tracestate` without `traceparent`;
    - other extensions not named `^[a-z0-9]{1,20}$` or not scalar;
    - control characters in attributes (C1 and DEL included);
    - duplicate member names at any depth;
    - integers beyond 2^53 − 1 in data;
    - lines that are not UTF-8.
  - `Parse` returns `*Error{Rule, Msg}` with the spec's rule codes. `Noncanonical` and `Canonical` handle the spool's form.
  - `NatsSubject()` is `anfs.<scope>.<type without io.>`, and `SubjectParts` reverses it. `Short()`, `FullType`, `NormalShort` and `IsNamespace` give the short type of SPEC.md §2.
  - `Line` takes a short or full type and an occurrence time. `AppendRecorded` writes `.bus.jsonl`.
- **The new subjects everywhere** (gap 23):
  - dispatch's `anfs.host.anfs.source.updated`;
  - `Rule.FilterSubject` adds the namespace: `report.*` gives `anfs.<id>.anfs.report.>`, `cnfs.*` gives `anfs.<id>.cnfs.>`, and a glob whose first token is a wildcard gives `anfs.<id>.>`;
  - the sink's `anfs.*.anfs.report.>`;
  - `on.type` and `anfsd match --type` are kept short in either form (gap 12).
- **The shipper** ships any layer's line. When a line's scope differs from its directory, it logs that once per file and ships the line by its source (gap 11).
- **`anfsd validate FILE...|-`** (gap 10):
  - It prints `FILE:LINE RULE message`. For a file in the spool's layout (both `.jsonl` and `.bus.jsonl`) it also checks `spool.scope` and `spool.month`.
  - `noncanonical` is reported but does not fail.
  - The last line gives the counts. Exit 1 when a line is invalid, 64 when no file is given.
- **The recorder** (gap 13, `internal/recorder`):
  - It is a durable consumer, `anfsd-recorder`, on `ANFS` (`anfs.>`, deliver all, explicit ack).
  - Each message is checked against the profile, and its subject against its event. An invalid or mismatched message gets one log line and is terminated.
  - If the event's id is already in its month's `.jsonl` or `.bus.jsonl`, the message is acked and nothing is written. This is also how a redelivery after a crash is skipped.
  - Otherwise the event's canonical form is appended to `<scope>/<YYYY-MM>.bus.jsonl` in one `O_APPEND` write, then the message is acked.
  - Ids are read once per month and then followed by offset.
- **`max_msg_size` 4202** on the stream (gap 14). NATS counts the headers in this limit (nats-server's `stream.go`: `len(hdr) + len(msg)`), so it is the body's 4095 bytes plus 107 bytes for the two headers every publish carries.
- **Run usage** (gap 15):
  - `harness.UsageOf` reads claude's stream (checked against 2.1.296's strings): the init line's model, the result line's `total_cost_usd`, and `modelUsage` summed over models. Without `modelUsage` it uses `usage`. Input counts input plus cache-read plus cache-creation tokens.
  - `run.completed` gets `model`, `input_tokens`, `output_tokens` and `cost_usd`. `run.failed` gets them when known.
  - The model falls back to the rule's.
- **Podman events at podman's time** (gap 16): `timeNano`, else `time`.
- README.md and AGENTS.md: the spec, the subjects, the recorder, `anfsd validate`, the usage, and the test-only dependency.

**core** (built by a sub-agent of this job; see "An incident" below):

- `events.sh` (gaps 17–20, 22):
  - The emitter checks `ANFS_ACTOR` against the actor pattern; when it fails, it warns and detects the actor instead. A detected actor that is invalid falls through to the next method, and finally to `shell`.
  - It refuses with a warning, exit 0:
    - any control character in subject, actor or spacepath;
    - DEL, C1 and non-UTF-8 bytes anywhere (no fork: glob replacement under `LC_ALL=C`);
    - an `io.anfs` type whose first segment is another layer's namespace.
  - The reader takes any layer's event (short type, scope).
  - `events ls` merges a month's `.jsonl` and `.bus.jsonl` in id order. `-f` follows both of the newest month's files, and `--since` compares the month alone.
  - The header points to the spec.
- `anfs events validate [SPEC...|FILE...|-] [--all]` (gap 22) runs `anfsd validate` and passes its output and exit status through. Without anfsd it exits 69, naming core/anfsd.
- `bus.sh` (gap 21): `ANFS_BUS_LAYERS` in nats.env (empty by default) makes one user per layer, with a password file each. In `server.conf` that user may publish `anfs.*.<layer>.>` and use reader's JetStream consumer API (`ANFS_BUS_CONSUMER_API`, one list), and subscribe to `anfs.>` and `_INBOX.>`.
- `anfs bus status`:
  - a `recorder` row (from `/jsz` with consumers);
  - the layer users named on the nats row;
  - a bug fixed: the shipper's lag counted `.bus.jsonl` files.
- `pins.yml` names v0.4.0, with no sums yet.
- The ui (gap 24):
  - Its consumer `ui` stays on `anfs.>` and takes the type from the body.
  - The Events page shows another layer's event by its short type, and its space from any layer's source.
  - Specs: the subjects use the new shape, and a stream that is removed and made again is recovered by itself.
- Fixtures: `packages/anfs/tests/fixtures/events-spec/`. Docs: AGENTS.md ("Events", "The bus", "Dispatch"), README, and the READMEs of nats, anfsd and the ui.

**docs**: this log. The pointer from 01's plan ("The envelope") and from AGENT-NATIVE.md ("Events and subscriptions") to anfs-io/events, which 05a left out.

## Commits (branch agents-05b, not pushed, not tagged)

| Repo | Commit | What |
| --- | --- | --- |
| runtime | `76d4165` | the profile for every layer, the vendored examples, conformance tests, the new subjects (gaps 1–9, 11, 12, 23) |
| runtime | `4e66b35` | `anfsd validate` (gap 10) |
| runtime | `21ee5fc` | the recorder; the stream's `max_msg_size` (gaps 13, 14) |
| runtime | `477de26` | run usage on `run.completed`/`run.failed` (gap 15) |
| runtime | `53ac5fa` | podman's events at podman's time (gap 16) |
| runtime | `4de9a25` | tests: the layer user from core's rendered conf; the recorder end to end; the e2e's ids |
| runtime | `e9f3196` | README, AGENTS.md |
| core | `7ef28ad` | events.sh to the spec, the vendored fixtures (gaps 17–20, 22's header) |
| core | `0d1e278` | core/nats: a publisher user per layer (gap 21) |
| core | `f048077` | `anfs events validate` (gap 22) |
| core | `8d1eb0a` | `anfs bus status`: the recorder row, the layer users, no lag for `.bus.jsonl`; pins v0.4.0 |
| core | `afa2c60` | the ui and the new subjects (gap 24) |
| core | `b743362` | AGENTS.md, README |
| docs | (this) | this log, the 01 and AGENT-NATIVE pointers |
| events | none | no spec bug needed a change (see "Spec notes") |

Versions: core's pre-commit hook bumps a version once per unpushed series.
- anfs stays 0.2.18 and core/anfsd stays 0.3.1: 02d already bumped them in the unpushed series.
- nats goes to 0.1.1 and the ui to 0.2.1.
- Runtime's release is v0.4.0, which `pins.yml` already names.
- **If main is pushed with 02d before this branch is rebased onto it**, the rebase does not run the hook. In that case, bump anfs and core/anfsd by hand.

## Decisions made during the work (deviations from the plan, and why)

1. **Rule codes beyond SPEC.md's table.** `Parse` names a rule for every refusal. Where the table has no code, it uses the spec's naming:
   - `json.parse`, `json.utf8`, `json.object` (as 05a's `tools/check` uses them);
   - `actorspace.form`, `actorrole.form`, `actorrun.form`, `spacepath.form`, `tracestate.form`;
   - `spool.scope` and `spool.month` (validate);
   - `subject.mismatch` (the recorder's log).

   These could join the table in a minor version of the spec.
2. **The conformance test asserts the rule, not only the refusal.** Each invalid example must fail with the manifest's rule, so the parser and the manifest cannot drift apart.
3. **`max_msg_size` is 4095 + 107.** NATS counts headers in the limit (as the plan suspected), so the limit is "4096 plus the two headers' room". A publisher that sends fewer headers can get up to about 45 more body bytes past the stream; the recorder checks the body itself.
4. **Lines misfiled in the spool.** A line whose scope directory or month file is not its own is shipped by its source (gap 11). The recorder then looks for it only in its own scope's month and records it there too, so it appears in the record twice. `anfs events validate` reports such lines (`spool.scope`, `spool.month`). None exist on this host.
5. **Usage test data is built, not recorded.** No harness may run here. `harness/testdata/claude-result.jsonl` uses 2.1.296's field names from its binary (`total_cost_usd`, `usage`, `modelUsage` with `inputTokens`, `cacheReadInputTokens` and the rest). The first real run is the live check. For opencode, pi and agy, anfsd's invocations print no usage, so they report none (`model` only, the rule's).
6. **An unregistered namespace's short type** (`jev.x.y`) reads as anfs's in `FullType`. SPEC.md §2 ties the short type to the registry, so a rule on such a type filters `anfs.<id>.anfs.jev.>`. Registering the namespace fixes that.
7. **By hand: the stream is removed before the new anfsd's first start**, not "right after the install" as plan step 6 says. Otherwise the new recorder (deliver all) reads the old stream first: every old message is on an old-shape subject, so it is terminated with one log line each. That is harmless but noisy. The order below avoids it.
8. **The e2e needs bash 5 on PATH**: core's `lib/anfs` needs it for the run that emits through `anfs_emit`, as 02d's run did. The bats ran with the pinned nats-server v2.15.1, nats v0.5.0, go, yq, bats and Homebrew's bash linked into the job's TMPDIR. The e2e's dispatch lines used a fixed id at the current time; they now derive the id from the time, since `time.id` refuses the old form.
9. **core** (sub-agent):
   - The fixtures are in `packages/anfs/tests/fixtures/events-spec/`, not `tests/fixtures/`: core keeps its tests per package.
   - Only `anfs events validate` was added; there is no `wsm events validate`.
   - The bash reader is not run over the `noncanonical` example: it needs the canonical form.
   - Reader's and the layers' JetStream permissions come from one list. The rendered text is the same as before.
10. **Layer users can delete any consumer on `ANFS`.** They get reader's `$JS.API.CONSUMER.*.ANFS.>`, which includes `CONSUMER.DELETE`. That holds for `reader` (the ui) already. It is acceptable on a dev host, but it is listed under open questions.
11. **New test dependency: santhosh-tekuri/jsonschema/v6 v6.0.3** (released 2026-06-28), as the plan names it, plus golang.org/x/text as an indirect dependency. They are used in tests only; the binary does not contain them.

## An incident (core's sub-agent)

While checking the layer-user rendering, the sub-agent ran `anfs_bus_nats_users` and `anfs_bus_nats_conf` with a temp `HOME` while `XDG_STATE_HOME` and `XDG_CONFIG_HOME` still pointed at the real ones.

- **What it wrote, twice:** the live `~/.local/state/nats/server.conf` (with cnfs and lgat users), `port` (same value), and `users/cnfs` and `users/lgat`.
- **What it restored:**
  - `server.conf` matches exactly what HEAD's `bus.sh` renders from the live settings; only its mtime changed.
  - `users/cnfs` and `users/lgat` are deleted.
- **Checked afterwards:** `users/` holds only `admin`, `anfsd` and `reader` (dated 2026-10-23), and `server.conf` has those three users.
- **The running nats-server was never reloaded**, so it never had the extra users.
- **The live passwords of `anfsd`, `admin` and `reader` were printed in the sub-agent's tool output**, which is part of this session's transcript. Rotating them is folded into the by-hand steps below (optional, recommended).
- Nothing else live was touched: no anfsd, no ui, no `launchctl`.
- Every later experiment ran under `env -i` with a temp `HOME`.

## Tests

- **Go** (`go test -race ./...` in `runtime/go`; gofmt and vet clean): **110 tests** (02d: 97). Three runs with `-count=3`: none flaky.
  - Conformance: all 64 examples through `Parse` (each invalid one with the manifest's rule) and through the schema, plus the canonical flag, the subject both ways and the two derived ids.
  - Rules the examples do not show; types and subjects; `Canonical`.
  - space: other namespaces' filters, `on.type` in both forms, a number in data never matches nor renders.
  - dispatch and the sink: on an old-shape subject, an event reaches neither.
  - shipper: cnfs nested data and lgat lines in the host directory (said once); `.bus.jsonl` is never read.
  - `anfsd validate`: output and exit codes, stdin, a spool directory's scope and month.
  - recorder:
    - one message at a time: recorded once and canonical; known again, and after a restart; spool lines known; invalid, mismatched and oversize messages terminated with one log line each;
    - on embedded JetStream: deliver all; the same event as two messages is recorded once; spool-shipped events are not recorded; an invalid message is never redelivered.
  - stream: a 4095-byte body is stored, and 4096 is refused with the headers.
  - The layer user from core's rendered conf: stores on `anfs.*.cnfs.>`; refused anfs's, lgat's and old-shape subjects and stream deletion; makes and reads its own consumer.
  - Usage; podman's time.
  - **Idle CPU**: `TestRunIdleAndStop` used 107 ms of CPU over ~11 s with the recorder running and acking all 11 spool lines, recording none (02d: 102 ms; limit 200 ms in 10 s). `TestRedispatchAndHeldIdle` passes.
- **e2e** (`go/test/e2e.bats`, the pinned nats-server v2.15.1 and nats v0.5.0, `ANFS_CORE_LIB` the job's core): **7 passing** (+1).
  - New: cnfs (core's layer user in the server.conf) publishes the same id twice, and it is in `m2vq7cfr4k/2026-10.bus.jsonl` once, as published.
  - The spool's lines are not recorded, `anfsd-recorder` delivers all and acks every message, `max_msg_size` is 4202, and cnfs is refused `anfs.<id>.anfs.report.filed`.
  - The existing tests pass on the new subjects with core's new `events.sh`.
- **events**: `tools/check` (check-jsonschema 0.38.2 from `tools/requirements.txt`, in a venv in the session's scratch directory): `64 examples (20 valid, 44 invalid): ok (manifest, line rules, subjects, derived ids, schema)`.
- **The real spool, read only**: `anfsd validate ~/.local/state/anfs/events/*/*.jsonl` (the job's build) reports `51 events, 0 invalid, 0 noncanonical (3 files)`.
- **core** (sub-agent; the last three cross-checked here: 73 passing):
  - `events.bats` 27 (+7)
  - `nats.bats` 12 (+1)
  - `packages/anfsd/tests` 34 (`bus.bats` 14, +1)
  - `settings`, `registry` and `rpc` pass
  - ui rspec 91 (+1), rubocop clean
  - `packages/anfs/tests/all` once: **756 passed, 0 failed**, 54 files, 58 s (pmm and pcm off; 02d: 747)

## Not run

- No harness, no real podman. Nothing installed, loaded or restarted. The live bus, anfsd and the ui were not touched, apart from the incident above.
- CI and the release: nothing was pushed or tagged. Linux was not run.

## Spec notes (events not changed)

- `actorclaimed`: SPEC.md says at most 256 **bytes**, but the schema's `maxLength` counts characters and `tools/check` does not check bytes. Go's parser counts bytes, as it does for `actor`. The SPEC table could say "yes (bytes: no)" for `actorclaimed.form`, as it does for `actor.form`, and `tools/check` could count bytes. This is not a bug in any example, so the repository was left as 05a built it.
- SPEC.md §6 lists `actorclaimed` before `actorspace` in "anfs writes"; 02d's `StampLine` writes it after `actorrun`. The order before `data` is free, so both are canonical. The sentence could say "e.g.".

## By hand, for the person, in this order

No migration code. The cursors are kept (`~/.local/state/anfsd/cursors.json` holds a byte offset per month file and does not depend on the stream), so nothing old is shipped again. The spool files are the record: all 51 lines on this host are valid.

1. **Merge.** Rebase `agents-05b` onto `main` in core after the `ppm-events` and `ui-sort` jobs. On the ui, the Events page's `event_log.rb` and `container.yml`'s version may conflict; take both. Then ff-merge in runtime, core and docs. events has no 05b changes.
2. **Stop anfsd and remove the old stream before the new anfsd ever starts** (deviation 7). Do it when no report is being filed.

   ```bash
   anfs bus stop anfsd
   anfs bus nats -- stream rm ANFS --force   # the stream and every consumer on it:
                                             # anfsd-dispatch, anfsd-sink-report, ui
   ```

   The ui's bus loop sees its consumer deleted, retries after 1 s, then 2, 5, 10, 30 and every 60 s ("cannot read stream ANFS") until the stream is back.
3. **Optional, recommended: rotate the bus passwords** (the incident). With anfsd stopped:

   ```bash
   rm ~/.local/state/nats/users/{anfsd,admin,reader}   # made again by step 5's core/nats install
   ```
4. **Build anfsd 0.4.0.** Either:
   - from source, as in 02d (ANFSD_BIN in anfsd.local.env):

     ```bash
     cd ~/spaces/anfs-dev/runtime/go && go build -o ~/.local/share/anfsd/dev/anfsd ./cmd/anfsd
     ```

   - or as a release: push runtime and tag `v0.4.0`, wait for the release, run `packages/anfsd/tests/pins --print` in core, put the sums into `packages/anfsd/pins.yml` and commit.
5. **Install.**

   ```bash
   anfs src update core && ppm install core/anfs core/nats core/anfsd
   ```

   - core/nats renders `server.conf` (new passwords if step 3 was done) and restarts NATS.
   - core/anfsd loads anfsd. On start, the shipper makes `ANFS` again (`max_msg_size` 4202).
   - anfsd-dispatch and anfsd-sink-report are made again (deliver new) with the new filters, and `anfsd-recorder` (deliver all) for the first time. Each is retried every second until the stream exists. `anfs bus logs anfsd` shows `consumer ...: ready` for all three.
   - Lines written while anfsd was stopped (the `source.updated` of `anfs src update`) ship on start, possibly before anfsd-dispatch exists, so they are not dispatched. The recorder sees them and finds them in the spool.
6. **The ui.** Its code changed (mounted, no new gem), and after step 3 the reader's password file did too:

   ```bash
   pcm down ui && pcm up ui
   pcm logs ui | grep bus             # "bus connected"
   ```

   Without the restart, the old ui makes `ui` again by itself within 60 s, with new events only.
7. **Check.**

   ```bash
   anfs bus status                    # nats, jetstream, anfsd, shipper, recorder (ok, 0 pending), dispatch: ok
   anfs bus nats -- stream info ANFS --json | jq .config.max_msg_size      # 4202
   anfs events validate --all         # "N events, 0 invalid, 0 noncanonical (M files)", exit 0
   ```

   After the next event (e.g. `anfs src update core`), run `anfs bus nats -- stream subjects ANFS`. It shows only new-shape subjects (`anfs.host.anfs.source.updated`, `anfs.<id>.anfs.…`). No `.bus.jsonl` appears while only the spool feeds the bus.
8. **When cnfs is ready to publish:** add `ANFS_BUS_LAYERS=cnfs` to `~/.config/nats/nats.local.env`, then run `ppm install core/nats`. That makes `users/cnfs` and renders the conf, and restarting NATS loads them. anfsd and the ui reconnect with their backoff. Then tell cnfs:
   - the password file `~/.local/state/nats/users/cnfs`;
   - the publish contract (SPEC.md §7.1);
   - the spec's commit until it is tagged.

   What cnfs publishes lands in `~/.local/state/anfs/events/<id>/<YYYY-MM>.bus.jsonl`, and `anfs events ls` shows it.
9. **After 05a's tag:**
   - copy the tagged `schema/` and `examples/` into runtime's `go/internal/cloudevent/testdata/spec/` and core's `packages/anfs/tests/fixtures/events-spec/`;
   - update both `VERSION` files to the tag;
   - run the Go tests and core's `events.bats`.

## Open questions

1. Layer users and `reader` may delete any consumer on `ANFS` (`$JS.API.CONSUMER.*.ANFS.>` includes `DELETE`). Should a layer user be narrowed to its own consumers? NATS cannot match a consumer-name prefix inside `$JS.API.CONSUMER.DELETE.ANFS.<name>` beyond a wildcard token, so the layer user's consumers would need a naming rule (`cnfs-*`) and an allow list per name.
2. The recorder logs one line per invalid or mismatched message (each is terminated, never redelivered), with no rate limit. A buggy direct publisher in a loop would flood anfsd's log. Should it get a per-minute count, as podman's drops have?
3. Rule codes the SPEC table lacks (deviation 1): add them in `v1.1.0`?
