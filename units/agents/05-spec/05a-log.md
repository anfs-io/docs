---
plan: agents/05-spec (05a)
job: agents-05a
date: 2026-10-24
---

# Log: agents/05a, the spec repo

## What was built

- **anfs-io/events** (new repository, branch `agents-05a` on the README-only `main`): the contract and nothing else.
  - `SPEC.md`: plan §2 in full, plus §3 as the conformance section. The event and its rules, types and namespaces with the short type, the space-id rule and per-layer sources, actors (asserted in development, attested in production, `actorclaimed`, `actorrun`), NATS subjects `anfs.<scope>.<namespace>.<rest>` on one stream, the spool and direct publishing (the publish contract, the publisher user per layer, the recorder), delivery, derived ids, extensions with tracing and `commitment`, versioning, and a table of rule codes.
  - `REGISTRY.md`: namespaces (`io.anfs`, `io.cnfs`, `io.lgat`; `io.rws`, `io.roteoh` reserved), extensions, and anfs's catalog from `ANFS_EVENT_TYPES` with each type's scope, subject, actor and data keys (read from core's `report.sh` and `sources.sh` and runtime's `supervise` and `podman`), and the run usage keys mapped to OTel's `gen_ai.*`.
  - `schema/event.v1.json`: JSON Schema draft 2020-12, `$id` `https://raw.githubusercontent.com/anfs-io/events/v1.0.0/schema/event.v1.json`.
  - `examples/valid/` (20), `examples/invalid/` (44), `examples/index.json` (`file`, `valid`, `rule`, `schema`, `subject`, `key`, `about` per example).
  - `tools/check` (Python 3, standard library) and `tools/requirements.txt` (check-jsonschema 0.38.2 and its dependencies, pinned by version and sha256).
  - `.github/workflows/ci.yml` (push and pull request; `actions/checkout` v7.0.1 and `actions/setup-python` v7.0.0 by commit; Python 3.12; a venv from the hashed requirements; `tools/check`).
  - `README.md`, `AGENTS.md`, `CHANGELOG.md`, `.gitignore`.

## Commits (branch agents-05a, not pushed, not tagged)

| Repo | Commit | What |
| --- | --- | --- |
| events | `32bf5ab` | SPEC.md, REGISTRY.md |
| events | `a5ce358` | the schema, the examples, the manifest |
| events | `f01bb0d` | tools/check, tools/requirements.txt, CI |
| events | `9037124` | README, AGENTS.md, CHANGELOG.md, .gitignore |
| docs | (this) | this log |

## How conformance is checked

- `tools/check` checks that the manifest lists every file under `examples/valid` and `examples/invalid` once and nothing else, with every field.
- It applies the line rules a schema cannot express (size with the newline, one line, unique members, `time` equal to the id's milliseconds, integers in data within 2^53 − 1, `actor` bytes, `actorclaimed` differing from the attested identity, the canonical form): a valid example breaks none (`noncanonical` is flagged exactly where the manifest says), an invalid one with `schema: false` breaks the rule it names.
- It computes each valid example's NATS subject from its source and type and compares it with the manifest's `subject`, and derives the id from each `key` and compares it with the example's id.
- It runs `check-jsonschema --check-metaschema` on the schema, then check-jsonschema on every example: valid ones pass, invalid ones with `schema: true` fail, invalid ones with `schema: false` pass.
- Result: `anfs work run agents-05a events -- env PATH=<venv>/bin:$PATH tools/check </dev/null` prints `64 examples (20 valid, 44 invalid): ok (manifest, line rules, subjects, derived ids, schema)`; `tools/check --no-schema` the same without the schema part.
- Also checked by hand: each `schema: true` invalid example fails the schema for its own rule and no other (check-jsonschema's JSON output, one error each); a mutated copy of the repository (a stray file, a wrong subject, a wrong key, a wrong `schema` flag, a wrong rule) gives one failure line each; the derived id recomputed by an independent Go program from SPEC.md §9's bit layout matches; every schema pattern compiles in Go's `regexp` and Ruby's `Regexp`.
- check-jsonschema ran from a venv in the session's scratch directory, made from `tools/requirements.txt` (`--require-hashes --only-binary=:all:`); nothing was installed into HOME or the job's directory.

## Decisions made during the work (deviations from the plan, and why)

1. **`actorrun` registered, and `actorclaimed`'s grammar taken from 02d.** The plan's table lists eight extensions and gives `actorclaimed` the actor's pattern. 02d (planned after 05's first draft) adds `actorrun` (the run id) and writes `actorclaimed` as `<actor>[ as <actorspace>/<actorrole>]` (`shell:roberto as tkj76y3nfx/person`), and says 05 registers both. The spec follows 02d: nine extensions; `actorclaimed` is `<actor>[ as [<space id>]/<role>]`, at most 256 bytes; it is present only when it differs from the attested identity written the same way (rule `actorclaimed.same`, a line rule).
2. **The actor's name excludes spaces and control characters explicitly** (U+0000–U+0020, U+007F–U+009F) instead of the plan's `[^\s]+`: `\s` means different things in ECMA-262, Go and Python, so the schema would not agree with itself across languages.
3. **Schema patterns hold control characters as JSON `\u` escapes**, not regex `\x` escapes: Ruby refuses `\x80`–`\x9f` in a UTF-8 regex ("invalid multibyte escape"). Checked in Go, Ruby and Python. SPEC.md notes that a Ruby implementation anchors patterns to the whole string.
4. **Extensions are `additionalProperties` plus `propertyNames`**, not `patternProperties` plus `additionalProperties: false`: `data`, `actor` and the other known names match `^[a-z0-9]{1,20}$` too, so a scalar-only pattern schema would have refused `data`. Same rules, expressed so they hold.
5. **Six invalid examples beyond the plan's list:** `missing-datacontenttype` (required in the table), `subject-control` (`attr.control`, U+0085), `actorclaimed-same`, `duplicate-member` (`json.unique`: a new MUST, since JSON parsers disagree on which duplicate wins), `commitment-value`, `commitment-namespace` (the schema's `commitment` rules). `two-lines` is one event written over two lines (what `line.one` refuses), not two events.
6. **More promoted lines.** runtime's `bash.jsonl` has three lines, not two: `report-filed`, `report-filed-test` and `source-updated` are its lines byte for byte. `report-closed`, `container-died`, `no-subject` (`podman.connected`, empty data) and `role-only` are lines from this host's spool, byte for byte.
7. **The 4096 bytes are the line with its newline**; the bus body is the line without it, so at most 4095 bytes (what `spool.Read` hands the shipper and `Parse` counts). SPEC.md says both.
8. **The manifest's fields are always present** (`null` where they do not apply) and carry an `about` string, so a consumer's struct is fixed. It has a top-level `version`.
9. **cnfs examples use space id `m2vq7cfr4k`**, not the plan's `k3x7q2mbzp`: runtime's `bash.jsonl` line 1, promoted as is, already uses `k3x7q2mbzp` with `spacepath` `anfs-dev`, and one id must not name two spaces in the fixtures. lgat's examples use `q7mwx4ne2r`.
10. **No LICENSE file.** None of anfs-io's repositories has one (core, runtime, docs, stack: none on disk, and GitHub reports no license for any of them), so "consistent with the others" is none. A public spec that other organizations vendor should have one; choosing it is the person's decision (see below).
11. **01's pointer is not added.** The plan's 05a lists "01's pointer" (a line in `units/agents/01-events/plan.md` naming the repository); this job's docs worktree was for this log only. It belongs with 05b's docs changes, or a one-line edit by hand.
12. **check-jsonschema's dependencies are pinned to versions at least two weeks old**: `certifi` 2026.7.22, not the resolver's 2026.10.23; the rest are the resolver's.

## By hand, for the person

1. Read `SPEC.md` (and `REGISTRY.md`).
2. Create `anfs-io/events` on GitHub (public, empty: `https://api.github.com/repos/anfs-io/events` is a 404 today). `space.yml` already lists it.
3. Decide the license (none of anfs-io's repositories has one); add `LICENSE` before tagging if wanted.
4. Merge `agents-05a` into `main` (`--ff-only`) in the events repository and in docs, push both, and check the first CI run.
5. `git tag v1.0.0 && git push origin v1.0.0` in events.
6. Then plan step 4: tell cnfs the tag (the gem can start on the fixtures), and 05b vendors `schema/` and `examples/` into runtime and core.
