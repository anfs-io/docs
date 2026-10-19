---
plan: stacks/01-stacks
job: stacks-01
date: 2026-10-19
---

# Log: stacks/01-stacks

## What was built

- `lib/anfs/stacks.sh` (new): the manifest (`stacks/<name>/stack.yml`; unknown key, bad entry or
  bad `status:` is 64), `stacks:` resolved depth-first with ppm's layering rule (cycle is 1 and
  names the chain), and install kind by kind (packages, skills, vaults, spaces, containers,
  machines) followed by the `status:` calls. A missing tool is 69, checked with the plan. Also
  `--build`, `-t host` only (anything else is 64, "plan 04"), the record, remove's rule, `list`
  and `show` (with `--json`), and `path`.
- `anfs`: `list|ls`, `show`, `install`, `remove|rm`, `path` at the top level. The old
  `cmd_install` is `src install`, unchanged except for its message. `anfs install <source>` is
  64 with the hint. It also sources `resolve.sh` now.
- `sources.sh`: `stacks` is in `GITSRC_DIRS`, so `src list` PROVIDES shows it. `ANFS_TOOLS` is
  unchanged.
- `sh/anfs.sh`: the shell reloads after `install`, `remove` and `src install`.
- Docs: core AGENTS.md has a new "Stacks" section, a `stacks/` row in the resource table, a top-level
  note in "The Command Line", and `anfs list` / `anfs show` in the key table. `anfs install
  <source>` is now `anfs src install` in README.md, packages/anfs/README.md, wsm/README.md and
  psm/AGENTS.md. There is no anfs skill.

## Commits (core, branch stacks-01)

- `afcb1e4` anfs: stacks (lib, commands, tests)
- `ab44ab4` docs: Stacks

## Deviations and decisions the plan left open

- **Vaults**: pvm has no `install` and does not read sources. `vaults:` is parsed and shown,
  but install says "pvm does not install vaults from sources yet", installs nothing and records
  nothing. Under `--build` it says "no vault is installed".
- **Spaces**: `wsm install` takes one level per run, so spaces get one `wsm install <level>` each
  instead of a single call for the kind. `status: cloned` adds `wsm repo install <level>` after
  the install. `installed` is what `wsm install` does by default.
- **Records for dependencies**: installing `b` (which depends on `a`) writes a record for each,
  and each record holds its whole closure. That is how "remove `b`, `a`'s resources stay" works:
  `a`'s record holds them.
- **`before`** is true only when the resource was there before this install *and* no other stack's
  record holds it. Without the second condition, a resource installed by stack X would become
  unremovable once stack Y also listed it.
- Install runs a tool only for the resources still missing. `status:` is always applied.
- **Remove**: if a tool fails, the record is kept (and this is reported), so remove can be run
  again. Removing a stack that is not installed is 1. A bare name with two records is 64.
- When two stacks declare the same resource, the first in resolution order (the dependency) wins,
  `status:` included.
- `show` prints `<stack> <version>: <description>` above the table, but not with `--json`. Its
  first rows are the dependency stacks (kind `stacks`, `actual` from their record). Its header
  `KIND RESOURCE DECLARED ACTUAL FROM` is in registry.bats's rule-5 list, and `STACK` is in the
  kinds list.
- `list` also shows installed stacks whose definition is gone (from the record).

## Tests

- `packages/anfs/tests/stacks.bats`: 21 tests, all pass. Stubs for ppm, psm, wsm, pcm and pmm
  log argv and answer `list --json` from a state file.
- `packages/anfs/tests/all`: 325 passed, 0 failed, 22 files (pmm and pcm off).
- By hand, in an isolated HOME (HOME, XDG_* and every `*_HOME` var under a temp dir) with the
  same stubs: `b` depends on `a`. Ran install `b`, list (both installed), show, then remove `b`.
  Only `b`'s package and container were removed, `a`'s were kept ("demo/a holds it"), `a.yml`
  stayed and `b.yml` was deleted.

## Not run / not verified

- No real tools: every tool call was checked against stubs only. Real `pcm stop`, `pmm stop`,
  `wsm repo install` and real `--json` output were not exercised end to end.
- pmm and pcm suites (off, per instructions). No VMs or boxes.
- No shellcheck run (not installed).
- `anfs remove system` / implode are plan 02. Targets other than host are plan 04.
