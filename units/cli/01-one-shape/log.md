---
plan: cli/01-one-shape
job: cli-shape
date: 2026-10-19
---

# Log — cli/01-one-shape

## What changed

| Tool | Before | After (`--json` keys) |
| --- | --- | --- |
| psm | `psm skills list\|remove\|prune` | `psm list` (`ls`), `remove` (`rm`), `prune`; `psm skills` exits 64 |
| `psm list` | `STATUS CONFIG SKILL REPO` | `SKILL STATUS` (`skill status config repo`; config and repo JSON only) |
| `psm repo list` | `CONFIG REPO NOTE` | `REPOSITORY STATUS CONFIG` (`repository status config`) |
| wsm | `wsm list -r` | `wsm repo list` (`ls`), `wsm repo install [SPEC] [-n]`; `list -r` exits 64 |
| `wsm list` | `ID DIR LEVEL NOTE` | `SPACE STATUS LEVEL DIR` (`space status level dir`) |
| `wsm repo list` | `REPOSITORY URL STATE` (`path url state`) | `REPOSITORY STATUS URL` (`repository status url`) |
| `pcm list` | `NAME SOURCE STATUS` | `CONTAINER STATUS` (`container status`) |
| `pmm list` | `NAME SOURCE IMAGE STATE ADDRESS` | `MACHINE STATUS IMAGE ADDRESS` (`machine status image address`) |
| `pmm image list` | `NAME SOURCE ARCH STATE KEY ...` | `IMAGE STATUS ARCH KEY ...` (`image status arch key ...`) |
| `pmm network list` | `NAME SOURCE KIND SUBNET STATE MACHINES` | `NETWORK STATUS KIND SUBNET MACHINES` |
| `anfs src list` | `ALIAS URL LIST STATUS PROVIDES` | `SOURCE STATUS URL LIST PROVIDES` |
| `anfs/pmm/ventoy iso list` | `ID NAME ARCH FILE STATE SIZE` | `ISO STATUS NAME ARCH FILE SIZE` (`iso status name architecture file size catalog`) |
| `anfs callbacks`, onboard, pvm (`ls`, `link ls`, `link validate`), `anfs work list` | `... STATE ...` | `... STATUS ...` (key `status`), same position |
| `anfs src add <url> local` | accepted | exits 64 (also when the alias comes from the URL) |

- `wsm repo install` walks the same levels as `list` (new `_each_level`), and for each installed
  one runs `_level_repositories`, the function install clones through.
- `lib/anfs/table.sh`: `anfs_sorted` (sort by a leading key field, drop it), so pmm's lists keep
  a name's rows (winner, then shadowed) together now that the first column is `source/name`.
- `registry.bats`: every `anfs_table` header in core's tools starts with a rule-1 kind or is a
  rule-5 list, and none has `STATE`.
- Readers changed: pvm (`wsm ls` `.space`) and its wsm stub; anfs-dev `anfs-work` (`wsm ls`
  `.space`, `wsm repo list` `.repository`), `container.sh` (`pcm list` `.container`), `vm.sh`
  (`pmm list` `.machine`, `.status`); core round trips (psm, pmm image/machine/network); pmm e2e.
- Docs: core CLAUDE.md "Lists and --json" (five rules, key table rewritten), "The Command Line"
  (verbs examples, a tool's main resource at its top level, `repo` group), wsm section; wsm
  README; psm AGENTS.md; cli.sh's example; anfs-dev dev README; ventoy README.

## Commits

- core (`cli-shape`): 776d46f anfs, e2bff20 psm, 97902fe wsm, afc4b82 pcm/pmm/pvm, bf37a42 round
  trips + e2e readers, d26b356 CLAUDE.md
- stack: 2ece926 ventoy
- anfs-dev: 4ded380 dev

## Deviations / choices

- `psm list`'s SKILL is `<config>/<skill>`: the source alias for an anfs source, the config file's
  name for a psm config (not `local/...`), so two configs naming one skill stay two rows.
- ISO ids stay unqualified: they are one namespace across catalogues (first catalogue wins), and
  `iso pull <id>` / `media: [id]` take the bare id.
- `psm repo list`'s REPOSITORY is the repo as declared (path or URL), not `source/name`.
- Rule 2 also applied to `anfs work list` (anfs-dev) and pvm's `link add` result key
  (`state` -> `status`), beyond the plan's table. A script reading `anfs work list --json .state`
  must read `.status`.
- Not touched: stack tools outside the plan (remotefs `STATE`, gem-local `NOTE`, opm, pikvm).

## Tests

- `packages/anfs/tests/all` (core): 303 passed, 0 failed, 21 files (pmm and pcm off)
- anfs-dev `bats packages/dev/tests`: 26/26
- stack `ventoy` (with `ANFS_TEST_LIB` = this worktree's lib/anfs): 18/18; `firstmate` has no bats
  (only a round trip)

## Changed, not run

- pcm `tests/sources.bats`; pmm `tests/{holds,image-build,image-sources,machine,media,network,sources}.bats`
  (suites off). By-hand in an isolated HOME (podman stubbed): `pmm image list`, `pmm list`,
  `pmm network list`, `pcm list` and their `--json` show the new headers and keys.

## Deferred

- `packages/pmm/tests/e2e debian`, the round trips (`roundtrip`, `roundtrip-macos`): readers
  updated, not run (no VMs).

## Not verified

- ventoy's own suite run against the installed lib/anfs (default) fails until core is installed.
