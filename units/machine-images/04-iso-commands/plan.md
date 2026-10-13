---
---

# Plan 04 — One `iso` command group

## Context — read these files first

- `packages/anfs/home/.local/lib/anfs/iso.sh` — the store, the downloader, the catalogue
- `packages/anfs/home/.local/lib/anfs/sources.sh` — `gitsrc_*` and "the `src` command": a command
  group a library defines and a tool adopts. The pattern used here
- `packages/anfs/home/.local/lib/anfs/cli.sh` — `cli_group`, `complete_<kind>`
- stack `packages/ventoy/home/.local/bin/ventoy` — `cmd_isos`, `cmd_get`, `get_one`
- `lib/pmm/media.sh` — `media:` ids; `lib/pmm/registry.sh` — `clean --isos` (was pim's)
- stack `isos/isos.yml` — the catalogue's fields

## Overview

Three tools share one ISO store and one catalogue, and each lets a person at it differently:

| Tool | Today |
| --- | --- |
| ventoy | `ventoy isos` lists the catalogue, `ventoy get <id>` downloads; `list`, `copy`, `remove`, `sync` are about the stick |
| pim | `pim clean --isos` deletes the ISOs its images name. Nothing lists or fetches |
| pmm | Nothing. A `media:` id is downloaded when a machine starts |
| anfs | Nothing, though the store, the catalogue and the hand-made `~/.config/anfs/isos.yml` are its |

After this plan there is one group, written once in `lib/anfs/iso.sh`, and every tool that uses
ISOs has it: `anfs iso`, `pmm iso`, `ventoy iso`. They are the same code, so they cannot drift,
and a tool that starts using ISOs later adopts it with one line.

## Implementation

### 1. The commands

```
<tool> iso list [--all]         the catalogue: ID NAME ARCH FILE STATE SIZE
<tool> iso show <id>            the entry, which catalogue file defines it, the file, its recorded sum
<tool> iso pull [-f] <id>...    download and verify into the store (alias: get)
<tool> iso add <url|file> [--id ID] [--name TEXT] [--arch ARCH] [--sha256 HEX | --checksum-url URL]
<tool> iso remove <id>...       delete the file from the store; --entry also drops a hand-made entry
<tool> iso path <id>            the file's path in the store, there or not
<tool> iso prune [-y]           delete files in the store no catalogue entry names
```

- `list` keys: `id`, `name`, `architecture`, `file`, `state` (`present`, `missing`, `unverified`:
  there, with no recorded sum), `size`, `catalog` (the file that defines the entry). Without
  `--all`, entries for another architecture than the host's are still listed: an amd64 ISO on an
  arm64 Mac is exactly what ventoy is for. `--all` adds files in the store no entry names.
- `add` writes to `$ANFS_CONFIG_HOME/isos.yml`, the hand-made catalogue, which is first in
  priority. A URL is recorded and not fetched (`pull` does that). A local file is copied into the
  store, hashed, and recorded with `checksum: sha256:<hex>`. An id that exists is refused (73)
  without `-f`.
- `remove` never edits a source's catalogue. Without `--entry` the entry stays and the file can
  be pulled again.
- `prune` asks through `anfs_confirm`.
- Exit codes are the toolkit's: 1 for an id that is not in the catalogue, 73 for an existing one.

### 2. Defined once

`lib/anfs/iso.sh` gains `anfs_iso_cli`: it declares `cli_group iso` and the commands above, and
defines `cmd_iso_list` and the rest, plus `complete_iso` (ids, from the catalogue files with no
yq: a line scan for `- id:`, to stay inside the completion budget).

A tool adopts it by calling `anfs_iso_cli` beside its own declarations. `anfs`, `pmm` and
`ventoy` do. The help text is the same in each; only the tool's name in the usage line differs,
which `cli.sh` already takes from `$CLI_TOOL`.

### 3. What each tool gives up

- ventoy: `isos` and `get` are removed in favour of `ventoy iso list` and `ventoy iso pull`.
  `get`'s `--keep`/copy-to-stick behaviour stays where it belongs, in `ventoy copy <id>`, which
  pulls when the file is missing.
- pmm: `image prune --isos` (pim's `clean --isos`) is removed; `pmm iso prune` and
  `pmm iso remove` replace it. `iso_names` goes with it.
- Images and `media:` keep fetching what they name when they need it. `iso pull` is for doing it
  ahead of time, which is what a build on a slow link or an offline rehearsal wants.

### 4. Documentation

CLAUDE.md: the commands under `lib/anfs/iso.sh` in "Lib Structure", the `iso list` row in the key
table, and one sentence in the pmm section. ventoy's README in stack.

## Test Spec

`packages/anfs/tests/iso.bats`, extended (local files over `file://`, as now):

- `list`: an entry from a source, one from the hand-made file shadowing it, `present`/`missing`/
  `unverified`; `--all` shows a stray file; `--json` keys.
- `add` a URL: recorded, not fetched. `add` a file: copied, hashed, pinned. An existing id: 73.
- `pull`: downloads and records the sum; a second run does nothing; a wrong pinned sum fails and
  leaves no file.
- `remove` deletes the file and keeps the entry; `--entry` on a source's entry is refused.
- `prune` removes only files no entry names, and nothing without `-y` or a terminal.
- `anfs iso list`, `pmm iso list` and `ventoy iso list` print the same rows.
- `registry.bats`' completion bound holds for `iso pull <TAB>`.

## Verification

- [ ] `anfs iso list`, `pmm iso pull debian-13-arm64`, `ventoy iso list` on this Mac
- [ ] `anfs iso add ~/Downloads/some.iso --id some`, then a blank machine with `media: [some]` boots it
- [ ] `ventoy isos` and `ventoy get` are unknown commands (64)
- [ ] `bats packages/anfs/tests </dev/null`, stack's ventoy tests
