---
---

# Plan 01 — One shape: the same groups, and every list alike

## Context — read these files first

- `units/cli.md`
- core CLAUDE.md: "Lists and --json" (the convention and the key table), "The Command Line"
- `packages/anfs/home/.local/lib/anfs/table.sh` — `anfs_row`, `anfs_table`
- Every caller of `anfs_table`: `grep -rn 'anfs_table "' packages` in core (anfs's `src list`,
  `callbacks`, `iso list`, onboard; pcm `sources.sh`; pmm `machine.sh`, `network.sh`,
  `registry.sh`, `snapshot.sh`, `sources.sh`; ppm `packages.sh`; psm; pvm; wsm)
- Every reader of a list: `grep -rn -- '--json'` in core, stack and `anfs-dev`; at the time of
  writing: core `packages/anfs/tests/roundtrip{,-macos}`, `packages/pmm/tests/e2e`, pcm's and
  pmm's `sources.sh`, psm, pvm and its test helper; `anfs-dev` `packages/dev/.../{container,vm}.sh`;
  stack `packages/firstmate` (the tool and its round trip)

## Implementation

### 1. psm: no `skills` group

`skills list|remove|prune` become `list` (alias `ls`), `remove` (`rm`), `prune`. `repo`,
`agents`, `install`, `sync`, `update`, `cd` and `implode` stay. Callers: the round trips
(`psm skills ls --json`), `packages/psm/tests`, `packages/psm/AGENTS.md`, core CLAUDE.md, and any
skill or README in core, stack and `anfs-dev` that names `psm skills`.

### 2. wsm: a `repo` group

`list -r|--repositories` goes.

```
wsm repo list [spec]       (ls) what -r printed
wsm repo install [spec]    clone what the installed levels' space.yml files declare and is
                           missing; nothing is stowed. The clone half of `wsm install`, through
                           the same function. It never pulls, as wsm never does
```

Callers: `packages/wsm/tests/{install,manage}.bats`, `packages/wsm/README.md`, the comment above
the list function in `wsm`, CLAUDE.md's key table.

### 3. Every list, one shape

1. **The first column is the resource**, headed by its kind in the singular, holding
   `source/name`: `PACKAGE`, `CONTAINER`, `MACHINE`, `IMAGE`, `NETWORK`, `SKILL`, `SPACE`,
   `REPOSITORY`, `SOURCE`, `ISO`. There is no separate `SOURCE` column. Its JSON key is the kind
   in lower case (`container`, `skill`, ...), the qualified value.
2. **`STATUS`** (key `status`) wherever a list says what state a thing is in, in the second
   column. `STATE`, and `NOTE` where it is a state, are renamed.
3. **`local` is a reserved source alias**: the tool's own config directory, the implicit source
   pcm, pmm and psm already have. `anfs src add` refuses it (64).
4. **A machine made by `pmm run`** has no definition, so no source: its first column is its
   plain name (`web1`). A defined machine is `core/web1`; a shadowed definition's row is its
   own `source/name`.
5. **Lists that are not of a source's resources** (`pmm ps`, `pmm snapshot list`, `anfs
   callbacks`, onboard's, pvm's) take rule 2 only.

| List | Today | Becomes |
| --- | --- | --- |
| `psm list` | `STATUS CONFIG SKILL REPO` | `SKILL STATUS`; config and the repo dir in `--json` only |
| `psm repo list` | `CONFIG REPO NOTE` | `REPOSITORY STATUS CONFIG` |
| `pcm list` | `NAME SOURCE STATUS` | `CONTAINER STATUS` |
| `pmm list` | `NAME SOURCE IMAGE STATE ADDRESS` | `MACHINE STATUS IMAGE ADDRESS` |
| `pmm image list` | `NAME SOURCE ARCH STATE KEY ...` | `IMAGE STATUS ARCH KEY ...` |
| `pmm network list` | `NAME SOURCE KIND SUBNET STATE MACHINES` | `NETWORK STATUS KIND SUBNET MACHINES` |
| `wsm list` | `ID DIR LEVEL NOTE` | `SPACE STATUS LEVEL DIR` |
| `wsm repo list` | `REPOSITORY URL STATE` | `REPOSITORY STATUS URL` |
| `anfs src list` | `ALIAS URL LIST STATUS PROVIDES` | `SOURCE STATUS URL LIST PROVIDES` |
| `anfs iso list` (and `pmm iso`, `ventoy iso`) | `ID NAME ARCH FILE STATE SIZE` | `ISO STATUS NAME ARCH FILE SIZE` |
| `ppm list` | `PACKAGE INSTALLED` | unchanged: already the shape |
| `pmm ps`, `snapshot list`, `anfs callbacks`, onboard, pvm | — | `STATE` → `STATUS` only |

An empty status is `-`, as any empty cell. Where a list had no status (`pcm list` for a service
never started), it says `-`.

### 4. Readers

Every reader found in Context is changed in the same series. Keys a reader used that are gone
(`name` + `source`, `state`) are read as the new ones (`container`, `status`). Nothing keeps the
old keys.

### 5. Documentation

CLAUDE.md "Lists and --json": the five rules above, and the key table rewritten. Each tool's
README and skill that shows a list.

## Test Spec

- Each tool's bats for its list: the header on a terminal (`script` or the helper's tty stub, as
  the suites do now), the first cell `source/name`, the `--json` keys.
- psm: `psm list`, `psm remove`, `psm prune` do what `skills ...` did; `psm skills` exits 64.
- wsm: `wsm repo list` is what `list -r` printed; `wsm repo install` clones a missing repository
  of an installed space and stows nothing; `wsm list -r` exits 64.
- `anfs src add <url> local` exits 64.
- `registry.bats`: every `anfs_table` header in the tools starts with a kind from rule 1 or is
  on rule 5's list (a grep over the sources, so a new list cannot drift).

## Verification

- [ ] `bats packages/{anfs,ppm,pcm,pmm,psm,pvm,wsm}/tests </dev/null` in core; stack's
      `ventoy` and `firstmate` suites; `anfs-dev`'s
- [ ] `packages/pmm/tests/e2e debian` (it reads `pmm list --json` and `pmm image list --json`)
- [ ] `grep -rn 'skills list\|skills ls\|list -r\|ls -r\|\.state\b'` in core, stack and
      `anfs-dev` finds nothing that means the old forms
