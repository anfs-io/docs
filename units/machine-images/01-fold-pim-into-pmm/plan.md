---
---

# Plan 01 — Fold pim into pmm

## Context — read these files first

- `packages/pim/home/.local/bin/pim`, `packages/pim/home/.local/lib/pim/*.sh` — everything moving
- `packages/pmm/home/.local/bin/pmm`, `packages/pmm/home/.local/lib/pmm/*.sh` — where it lands;
  `machine.sh` and `settings.sh` are the readers of `pim builds --json`, `pim path`, `pim hold`
- `packages/anfs/home/.local/lib/anfs/cli.sh` — `cli_group`, two-word commands
- `packages/anfs/home/.local/lib/anfs/catalog.sh` — one catalogue per process
- `packages/anfs/home/.local/lib/anfs/paths.sh` — `anfs_tool_dirs`
- `CLAUDE.md` — "pim: VM images", "pmm: machines", "Lists and --json", "The Command Line"

## Overview

Move pim's commands under `pmm image`, merge its libraries into `lib/pmm`, move its data under
pmm's directories, replace holds with a scan, and delete the pim package. How a build works does
not change in this plan; where the command and the files are does.

`pmm image` is the reference: `pcm image`, when it is written, takes its verbs, its list keys and
its `run <image>` from here. It is not a port of pim's command list. What was there only because
pim was a tool of its own is dropped.

## Implementation

### 1. Commands

```
pmm image list [image]         every image, one row per built arch: is it built, is the build current
pmm image show <image>         the definition, what a build runs, its builds
pmm image build <image>...     build unless current   [-f] [-v] [--arch A] [--console] [--test]
pmm image test <image>         boot a throwaway copy of the build and run the image's test.sh
pmm image export <image>       write the build as one standalone qcow2   [-c] [-o FILE]
pmm image remove <image>...    delete its builds; the definition is kept
pmm image prune                delete superseded builds nothing is on
pmm image new <name>           (plan 02)
pmm image path|cd|validate     as every kind has
```

| pim had | Now | Why |
| --- | --- | --- |
| `list` and `builds` | `image list` | Two lists of the same things. `list` said "built or not"; `builds` gave the disk and key a reader needs. One list has both |
| `verify` | `image test` | `validate` checks a definition without booting anything; `verify` booted the build and ran a script. The names said the same thing for two different jobs. `verify.sh` becomes `test.sh`, `--verify` becomes `--test` |
| `publish` | `image export` | "Publish" is the deploy tool's word |
| `clean` | `image prune` | docker's word for the same thing |
| `install` | `pmm install` | One install per tool: `pmm install source/` builds its images, then networks, then machines. `PMM_INSTALL=validate\|create` as now |
| `hold`, `release` | gone | step 4 |
| `implode` | `pmm implode` | |

`pmm image list` keys, which replace both old lists for every reader (`pmm` itself no longer
needs them; the deploy tool does): `name`, `source`, `arch` (null: not built), `state` (`built`,
`stale`: the definition changed since, `unbuilt`, `shadowed`), `key`, `disk`, `backend`, `user`,
`sshkey`, `held` (a list), `built`, `tested` (true when this build passed `image test`).

Declared with `cli_group image` and `cli_cmd "image build" ...` in `lib/pmm/commands.sh`.
`complete_image` already exists in pmm's completion for `--image`; it serves both.

`pmm migrate`, `lib/pmm/migrate.sh` and `lib/pim/moved.sh` are deleted: there is nothing to migrate.

### 1a. `run` is from an image, `up` is a defined machine

Today `run` is an alias of `up`, and a machine from an image is `pmm up --image debian-13`. With
images in the same tool the two words can mean what they mean in docker and compose:

```
pmm run <image> [--name NAME] [--cpus N] [--memory SIZE] [--disk SIZE] [--share PATH] [-p H:G]
pmm up [machine...]
```

- `run` creates a machine from the image and starts it. Without `--name` it is the image's name,
  then `<image>-2`, `<image>-3`: a second `run` is a second machine, not "already running".
- **`run` writes no definition.** Today they write
  `~/.config/pmm/machines/<name>/machine.yml`, which puts a tool's record of a throwaway machine
  in the directory for what a person configured, and makes it look like something to keep. The
  settings from the command line (cpus, memory, disk, shares, ports, user) go into the machine's
  own `state.yml`, beside its disk in `$PMM_DATA_HOME/machines/<name>/`, where the image, build
  key and ssh key already are. They live exactly as long as the disk and go with `pmm remove`.
- `up`, `down`, `reset`, `shell` and `remove` read `state.yml` for such a machine. A machine
  with a definition reads its settings from the definition, as now.
- `pmm list` shows it with no source and the state it is in; `pmm show` prints its settings.
- `run --rm` removes the machine when it is shut down.
- `~/.config/pmm/machines/` holds only definitions a person wrote, like any source's `machines/`.
  CLAUDE.md's "Every machine has a definition" becomes "a machine is defined or was run from an
  image".
- `up` starts machines that are defined, or that an earlier `run` or `create` made. `--image`
  leaves `up`, and `create` goes (step 1b).

### 1b. The whole of pmm after this unit (to be agreed before coding)

Besides the `image` group (step 1) and the `iso` group (plan 04):

```
Machines
  pmm run <image> [--name N] [--rm] [--no-start] [--profile P] [--cpus N] [--memory SIZE]
          [--disk SIZE] [--share PATH] [-p H:G] [--media M] [--user U] [--arch A]
                              a machine from an image, started. With --media and no image: blank
  pmm up [machine...] [--console] [--display] [--media M] [--share PATH] [-p H:G]
                              start machines: a defined one is made first if it is not yet
  pmm down <machine>...       shut down (alias: stop)
  pmm shell <machine>         log in (alias: ssh)
  pmm exec <machine> -- <cmd>...      run a command in it
  pmm ps                      running machines
  pmm list                    every machine: defined, made, running (alias: ls)
  pmm show <machine>          settings, the build it is on, snapshots
  pmm reset <machine> [snapshot]      back to a snapshot, or onto its image's current build
  pmm remove <machine>...     stop and delete the machine and its snapshots (alias: rm)

Definitions
  pmm path|cd <machine>       the definition directory
  pmm validate [machine...]

Snapshots
  pmm snapshot list <machine>
  pmm snapshot create <machine> [name]
  pmm snapshot remove <machine> <name>

Networks (Linux)
  pmm network create <name> --kind K [--subnet CIDR] [--interface BR] [--domain D]
                              make a network now; recorded with pmm's data, no definition
  pmm network list|show|path|install|remove

The toolkit's
  pmm install <spec>...       what `anfs install` runs (below)
  pmm implode
```

What changes from today, beyond `run`:

| Today | After | Why |
| --- | --- | --- |
| `create [--image]` | gone: `run --no-start` | `create` did two jobs: make a machine without starting it, and write a definition. The first is a flag on `run`. The second is not a command any more (below) |
| `snapshot <machine> [name]`, no name lists | the `snapshot` group | A command that lists or creates depending on an argument, and no way to delete one. A group has the same verbs as every other |
| `remove` deletes a local definition | `remove` never touches a definition | A machine made by `run` has none. One a person wrote in `~/.config/pmm/machines/` is theirs to delete |
| `network create` writes a definition and makes the network | `network create` makes it and records it in `$PMM_DATA_HOME/networks/` | What `podman network create` does: its record is in podman's storage, not in a config directory |
| `shell <machine> [-- cmd]` | `shell` and `exec` | pcm has the pair; one API for both tools (`container-images` unit). `shell <machine> -- cmd` keeps working as `exec` |
| `up --image` | `run` | step 1a |
| `migrate` | gone | |

**No command writes a machine or network definition.** `run` and `network create` make the
thing; a definition is a file of a few lines that a person writes when they want the thing kept
in a source (or in `~/.config/pmm/`). A second command that does nearly what the first does, and
differs only in where it leaves a record, is one too many. `pmm show <machine>` and
`pmm network show <network>` print the settings as the YAML a definition holds, so turning
something made on the fly into a definition is a copy. `image new` is different and stays: an
image is a directory of files, and it scaffolds them.

**`pmm install <spec>`** is the verb every tool has for `anfs install <source>`; a person uses
`run` and `up`. `pmm install acme/` builds that source's images, installs its networks, then
makes and starts its machines. `pmm install acme/web1` does the same for one machine and what
it needs. `PMM_INSTALL=create` stops before starting anything and `PMM_INSTALL=validate` only
validates (the round trips use both). It stays, unchanged apart from building images first.

### 2. Libraries

pim's files move into `lib/pmm/`. Where a name collides (`sources.sh`, `validate.sh`,
`commands.sh`, `completion.sh`, `implode.sh`) the image side goes in under an `image-` prefix
(`image-sources.sh`, `image-validate.sh`); the rest keep their names (`build.sh`, `qemu.sh`,
`ssh.sh`, `tart.sh`, `template.sh`, `verify.sh`, `registry.sh`, `iso.sh`, `anfs.sh`, `core.sh`,
`defaults/`).

- pmm's sourcing of `~/.local/lib/pim/{core,qemu,ssh,tart}.sh` becomes ordinary sourcing of its own
  directory, and the "must not define a `cmd_` function" rule is dropped from CLAUDE.md.
- What configures images is `PMM_IMAGE_*`: `PMM_IMAGE_INSTALL_TIMEOUT`, `PMM_IMAGE_SSH_TIMEOUT`,
  `PMM_IMAGES_HOME` (the local definitions). What the machine side reads too is plain `PMM_*`,
  because it is not about images: `PMM_KVM_DEV`, `PMM_FW_CODE`, `PMM_FW_VARS`, `PMM_SSH_HOST`.
  `PIM_INSTALL=validate` becomes part of `PMM_INSTALL`. What a provision step sees
  (`PIM_IMAGE`, `PIM_ARCH`, `PIM_USER`, `PIM_FILES`) is renamed in plan 03 with the contract.
- The image catalogue and the machine catalogue cannot be loaded together (`catalog.sh`). pmm
  already loads profiles and machines in turn; `create` resolves the image first, keeps what it
  needs (id, directory, build key, disk, user, key file) in variables, then loads machines.
- The local image source moves: `~/.config/pim/images/` becomes `~/.config/pmm/machines/images/`.

### 2a. One tool, one directory in a source

Everything pmm reads from a source goes under `machines/`:

```
machines/
  <name>/machine.yml        a machine
  profiles.yml              shapes
  images/<name>/image.yml   an image             (was images/<name>/ at the top)
  networks/<name>/network.yml                    (was networks/<name>/ at the top)
```

- The rule becomes: a tool owns one top-level directory of a source (`packages/` ppm,
  `containers/` pcm, `machines/` pmm, `skills/` psm, `spaces/` wsm; `isos/` is anfs's, shared).
  `containers/images/` is kept for container images as definitions of their own
  (`container-images` unit), so the two tools have the same layout.
- `images` and `networks` are not machine names: `validate` and `run --name` refuse them. The
  marker files already keep the catalogues apart (`machine.yml`, `image.yml`, `network.yml`), and
  `catalog_load` takes `machines/images` as a kind as it is.
- `anfs_tool_dirs` goes: with one directory per tool, `anfs_tool_dir` is enough. The install
  order (images, networks, machines) is pmm's own, in `pmm install`.
- `anfs src list`'s `provides` shows `machines` for a source with any of the three.
- `~/.config/pmm/` mirrors it: `machines/`, `machines/images/`, `machines/networks/`.
- Core's `images/` moves, with any other source's and the acme fixture's.

### 3. Data

| Was | Now |
| --- | --- |
| `$PIM_DATA_HOME/images/<name>/<arch>/` | `$PMM_DATA_HOME/images/<name>/<arch>/` |
| `$PIM_DATA_HOME/keys/<name>/` | `$PMM_DATA_HOME/keys/<name>/` |
| `$PIM_DATA_HOME/published/` | `$PMM_DATA_HOME/exports/` |
| `$PIM_STATE_HOME/{build,logs}` | `$PMM_STATE_HOME/{build,logs}` |
| tart VMs `pim-<name>-<key>` | `pmm-image-<name>-<key>` (machines stay `pmm-<name>`) |

Nothing is moved by code. The log entry says to run `pim implode -y` (and delete tart's `pim-*`
VMs) before installing this version. `_iso_adopt` (the move out of `$PIM_CACHE_HOME/isos`) is
deleted.

### 4. Holds become a scan

`holds/<key>/<holder>` files and `pim hold|release` go. What keeps a build is derived when it is
asked for: `image remove` and `image prune` read every machine's `state.yml` and every
snapshot's record for the build key they are on. `image list`'s `held` key is filled the same way.

### 5. One fix that belongs here

`qemu_accel` falls back to TCG silently when a guest of the host's own arch cannot use the
accelerator (Linux: `/dev/kvm` missing or not writable, which is what a user outside the `kvm`
group gets). The build then takes ten times as long with only `(tcg)` in one line to say why.
`image build` and `up` warn once, saying what is missing and how to get it
(`ppm install core/pmm-network` adds the group, or `usermod -aG kvm`). A foreign-arch guest is
TCG by design and does not warn.

### 6. Everything that names pim

- `packages/pim/` is deleted. `packages/pmm/package.yml`: `depends: [qemu]`, and the tart tap
  (`brew: {macos: [openai/tools/tart]}`) with its comment.
- `packages/anfs/package.yml` (depends), `bin/anfs` (the kinds table, `install` order, `implode`
  order), `anfs.conf` (`PIM_*` settings), `paths.sh`, comments in `catalog.sh` and `iso.sh`.
- `packages/dev/home/.local/lib/ppm/vm.sh` (`ppm vm build`).
- `packages/anfs/tests/roundtrip`, `roundtrip-macos`, `bash.bats`, `catalog.bats`, `iso.bats`,
  `fixtures/acme/README.md`.
- stack: `packages/ventoy` (a hint that names `~/.cache/pim/isos`; it is `~/.cache/anfs/isos`),
  `docs/pdt.md`.
- `CLAUDE.md`: the pim section folds into pmm's; the layout, lib structure, testing and key tables.

### 7. Tests

pim's bats suites move to `packages/pmm/tests/` (`image-build.bats`, `image-sources.bats`,
`image-validate.bats`, `qemu.bats`, `template.bats`); `holds.bats` is rewritten against the scan.
pmm's helper no longer installs "the real pim" beside the stubs. `packages/pim/tests/e2e` and
`packages/pmm/tests/e2e` become one script with the image half first.

## Test Spec

- Every moved suite passes unchanged apart from command names.
- `image remove` refuses an image a machine is on, and one a snapshot is on; `-f` removes it.
- `image prune` keeps a superseded build a machine is still on.
- With `PMM_KVM_DEV` pointed at an unwritable file on Linux, `image build` warns once.
- `image list`: an unbuilt image is one row with a null arch; a built one a row per arch; a
  changed definition is `stale`; `--json` has every key above.
- `run debian-13` twice makes `debian-13` and `debian-13-2`; `--name web1` makes `web1`; a name
  that exists is refused (73). `up --image` is refused as an unknown option.
- After `run --cpus 4 --share /tmp/x debian-13`: nothing is written under `$PMM_CONFIG_HOME`;
  `down` then `up` starts it with 4 cpus and the share; `remove` leaves no trace. `run --rm` then
  `down` leaves no machine.
- `pmm image <TAB>` completes sub-commands; `pmm image build <TAB>` completes image names under
  100 ms (`registry.bats`' bound).

## Verification

- [ ] `command -v pim` finds nothing after a fresh install; `~/.local/lib/pim` does not exist
- [ ] `pmm image build debian-13` then `pmm run debian-13` then `pmm shell debian-13 -- true`
- [ ] `pmm implode -y` leaves no `pmm` directory in any XDG home and no `pmm-*` VM in tart
- [ ] `bats packages/pmm/tests </dev/null`, `packages/pmm/tests/e2e debian`
- [ ] `packages/anfs/tests/roundtrip debian`, `roundtrip-macos`
- [ ] CLAUDE.md has no pim section and no stale key table rows
