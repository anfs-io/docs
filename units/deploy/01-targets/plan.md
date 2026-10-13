---
---

# Plan 01 — Targets, the tool, the provider contract

## Context — read these files first

- `packages/pcm/home/.local/bin/pcm` and `lib/pcm/{commands,sources,validate}.sh` — the smallest
  complete tool built on `catalog.sh` and `cli.sh`; copy its shape
- `packages/pcm/home/.local/lib/pcm/edges.sh`, `edges/*` — the provider pattern, and how a
  provider runs with credentials (fnox)
- `packages/pcm/home/.local/lib/pcm/compose.sh` — how a service's `.env.schema` is resolved
- `packages/anfs/home/.local/lib/anfs/{catalog,cli,table,paths,bash,log}.sh`
- `packages/anfs/home/.local/bin/anfs` — `install` and `implode` order; `anfs_tool_dirs`
- `packages/anfs/tests/fixtures/acme` — the fixture source every kind is added to
- `CLAUDE.md` — "The Command Line", "Lists and --json", exit codes

## Overview

A new package `core/pdm`: the command, `targets/` as a kind found in every source, and the
contract a provider meets. No provider that reaches a real system yet; a `null` provider that
records what it was asked is enough to build and test the tool.

## Implementation

### 1. The kind

```yaml
# targets/<name>/target.yml
kind: proxmox              # the provider
description: The lab PVE node
# everything else is the provider's: passed to it as k=v, validated by its `check`
endpoint: https://pve1.lab:8006
node: pve1
storage: local-lvm
bridge: vmbr0
ssh: root@pve1.lab
```

- `catalog_load targets target.yml local "$PDM_CONFIG_HOME/targets"`. Targets do not layer: the
  highest-priority source's definition is the target (`ANFS_RESOLVE_FIRST`).
- `.env.schema` beside it declares the target's credentials, resolved the way pcm resolves a
  service's. A provider runs with them in its environment and never reads the file itself.
- `anfs_tool_dirs pdm` is `targets` (plan 03 adds `deployments` after it).

### 2. Commands

```
pdm target list|show|path|cd|validate
pdm target install <spec>      validate, and run the provider's check against the real place
pdm target remove <target>     forget what pdm recorded about it; refuses while publications exist
pdm install <spec>             what `anfs install` runs
pdm implode
```

`pdm target list` keys: `name`, `source`, `kind`, `state` (`ready`, `unchecked`, `failing`,
`shadowed`), `note`. Added to CLAUDE.md's table.

### 3. The provider contract

A provider is `$PDM_CONFIG_HOME/providers/<kind>`, else `~/.local/lib/pdm/providers/<kind>`.
It is an entry point (`bash.sh` first). The target's keys other than `kind` and `description`
arrive as `k=v` arguments after the verb's own; credentials are in the environment.

| Verb | Does |
| --- | --- |
| `check [k=v...]` | Prints what is wrong (a missing key, a credential, an unreachable host) and fails. Offline checks only with `PDM_OFFLINE=1`, which is what `validate` sets |
| `accepts` | Prints the artifact kinds it takes, one per line: `machine-image`, `container-image` |
| `publish <kind> <name> <arch> <key> <file> [k=v...]` | Puts the artifact on the target; prints its remote id on the last line. Idempotent for the same key |
| `unpublish <remote id> [k=v...]` | Removes it; idempotent |
| `published [k=v...]` | Prints what the target holds that pdm put there: `remote id<TAB>name<TAB>arch<TAB>key` |
| `tfvars [k=v...]` | Prints `name=value` lines pdm exports as `TF_VAR_*` for a deployment (plan 03) |

A provider marks what it publishes (a tag, a description field) with name, arch and key, so
`published` is answered by the target, not from pdm's state. pdm's state is a cache of it.

A kind with no provider fails `validate`: "nothing provides proxmox targets: install the package
that does".

### 4. The `null` provider

`providers/null`: `publish` copies the file into the directory the target names (`dir:`) and
prints its path; `published` lists that directory. It is what the bats suites and the acme
fixture use, and it is a working "publish to a directory or a mounted share".

### 5. `pdm publish` (the tool's half)

```
pdm publish <image> --to <target> [--arch ARCH] [-f]
pdm unpublish <image> --from <target> [--arch ARCH]
pdm published [target]
```

- Reads `pmm image list --json` for the image's current build (`key`, `arch`, `backend`).
  No build: exit 1 saying to run `pmm image build`. pdm never builds.
- Asks the target (`published`) whether that key is there. If so, done. `-f` republishes.
- `pmm image export -o <tmp>` for a standalone file, hands it to `publish`, records
  `$PDM_STATE_HOME/published/<target>/<name>-<arch>.yml` (`key`, `remote_id`, `at`).
- A newer build is published beside the old one; the old one is removed only when no deployment
  records it (plan 03). Until then it is listed with `superseded`.
- `pdm published` keys: `target`, `image`, `arch`, `key`, `remote_id`, `state`.

### 6. Package and wiring

- `packages/pdm/`: `package.yml` (no `depends: [pmm]`: pdm degrades to "no artifact tools" and
  says so; `anfs` depends on both), `home/.local/bin/pdm`, `home/.local/lib/pdm/`,
  `home/.config/zsh/pdm.zsh` for `cd`, `tests/`.
- `packages/anfs`: depends, the kinds table in `bin/anfs`, `install` order (after pmm and pcm:
  it consumes their artifacts), `implode` order (before them).
- `fixtures/acme` gains `targets/dropbox/` (a `null` target) and the round trips check it with
  `PDM_INSTALL=validate`.

## Test Spec

- Catalogue: local shadows a source; a shadowed target is listed and nothing else.
- `validate`: unknown kind; missing `kind`; a provider `check` that prints.
- `publish` to a `null` target: first run copies and records; second run does nothing; after the
  build key changes it publishes again and marks the old one `superseded`; `-f` republishes.
- `publish` with no build exits 1 and does not call the provider.
- A provider in `$PDM_CONFIG_HOME/providers/` wins over a stowed one of the same name.
- `--json` for each list; `registry.bats`' completion bound holds for `--to <TAB>` (file names
  only: no yq, no provider call).
- `implode` removes state and leaves the target's own contents alone.

## Verification

- [ ] `bats packages/pdm/tests </dev/null`
- [ ] `pdm target list`, `pdm publish debian-13 --to dropbox`, `pdm published`
- [ ] `packages/anfs/tests/roundtrip debian` with the acme target
- [ ] CLAUDE.md: the kind, the contract table, the key table
