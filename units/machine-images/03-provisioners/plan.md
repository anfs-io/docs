---
---

# Plan 03 — Provisioners

## Context — read these files first

- `lib/pmm/build.sh` — `_build_qemu`'s provision loop, `_provision`, `build_key`
- `lib/pmm/ssh.sh` — `run_script`, `push_dir`
- `lib/pmm/tart.sh` — `_tart_root`: the same step on a guest whose sudo wants a password
- `lib/pmm/anfs.sh` — the `anfs:` key, today a special step in the build
- `packages/pcm/home/.local/lib/pcm/edges.sh` and `edges/*` — the provider pattern being reused
- `packages/anfs/home/.local/lib/anfs/bash.sh` — every provider is an entry point

## Overview

A provision step is run by a provisioner: an executable that is given a way to reach the guest
and one step, and does whatever it does. pmm ships two, `sh` and `anfs`. Anything else is a file
another package stows; pmm does not change to gain it.

Two things are fixed now because they cannot be added later without breaking images: where a
step runs (the provisioner decides: `sh` pipes the script into the guest, ansible would run on
the host against it), and what a step's inputs are (only the provisioner can say, and the build
key needs them).

## Implementation

### 1. Finding one

The provisioner of `provision/<NN>-<name>.<ext>` is `<ext>`. It is the first of:

```
$PMM_CONFIG_HOME/provisioners/<ext>
~/.local/lib/pmm/provisioners/<ext>       pmm's own, and what packages stow
```

`validate` fails a step whose provisioner is not found ("nothing provides .yml steps: install
the package that does") and runs each provisioner's `check`.

### 2. The contract

Called with a verb, like a pcm edge:

| Verb | Does |
| --- | --- |
| `check` | Prints what is missing on the host (a binary, a version) and fails; nothing printed is ready |
| `fingerprint <step>` | Prints, one per line, the files the step's result depends on, the step first. pmm hashes them into the build key. A provisioner whose inputs are a tree (roles, vars) lists the tree |
| `run <step>` | Runs the step against the guest. stdout and stderr are the step's log; a non-zero exit fails the build |

`run` gets the guest in its environment. It is given a way to run something in the guest, not
an ssh session, so the same provisioner works wherever the guest is:

| Variable | |
| --- | --- |
| `ANFS_IMAGE_GUEST_EXEC` | A command line that runs its arguments as root in the guest, stdin passed through. `sh` needs nothing else |
| `ANFS_IMAGE_GUEST_PUSH` | A command line that copies a local path to a guest path |
| `ANFS_IMAGE_GUEST_TRANSPORT` | `ssh` today. With `ssh`, also `ANFS_IMAGE_GUEST_HOST`, `_PORT`, `_USER`, `_KEY`, for a provisioner that brings its own connection (ansible) |
| `ANFS_IMAGE_NAME`, `ANFS_IMAGE_ARCH`, `ANFS_IMAGE_BACKEND`, `ANFS_IMAGE_DISTRO` | what is being built |
| `ANFS_IMAGE_USER` | the image's user |
| `ANFS_IMAGE_DIR` | the definition directory |
| `ANFS_IMAGE_FILES` | where `files/` is in the guest, empty when the image has none |

`ANFS_IMAGE_GUEST_EXEC` hides the one thing that differs between backends. A qemu guest's user
has passwordless sudo. A tart guest's stock account needs its password; pmm installs an askpass
helper before the first step and removes it in finalize, so a provisioner never sees the
password and does not ask which backend it is on.

pmm keeps: ordering, pushing `files/`, the log (`_provision`: timing, `-v`, the tail on failure),
and finalize. A provisioner is only `run`.

### 3. `sh`

`provisioners/sh`: `fingerprint` prints the step; `run` is today's `run_script` (the script piped
to `$ANFS_IMAGE_GUEST_EXEC env ... bash -s`). The script sees the `ANFS_IMAGE_*` variables above
except the `GUEST` ones (was `PIM_IMAGE`, `PIM_ARCH`, `PIM_USER`, `PIM_FILES`), and the shipped
images' steps are updated. `PIM_SSH_PUBKEY` for tart images becomes `ANFS_IMAGE_SSH_PUBKEY`.

### 4. `anfs`

The `anfs:` key leaves `image.yml` and `lib/pmm/anfs.sh` leaves the build. A step file
`provision/50-toolkit.anfs` holds what the key held:

```yaml
sources: host            # host | pushed
repo: <git url>
packages: [stack/zsh]
```

`provisioners/anfs`: `run` is today's `build_anfs_step` (copy the host's sources or fetch
install.sh, run it as the user). `fingerprint` prints the step, and for `sources: host` a file
it writes with each source's commit and dirty-state hash, which is what
`anfs_host_fingerprint` put in the key. `machines/images/anfs-dev/` is converted.

It is the proof that the seam carries a provisioner that runs on the host and has inputs
outside the image directory.

### 4a. The same step in a container build (not built here)

A step that installs an application stack is the same work whether the result is a VM image or
a container image. The contract above allows it without knowing: a container build would set
`GUEST_EXEC` to `buildah run <container> --` and `GUEST_TRANSPORT` to `buildah`, and `sh` would
run unchanged. One thing would have to change when pcm wants it, and it is a move:

- The variables a step sees are already tool-neutral (`ANFS_IMAGE_*`, decided 2026-10-13), so a
  step written today runs in either build. pmm's own settings (`PMM_IMAGE_INSTALL_TIMEOUT` and
  the like, in `anfs.conf`) keep pmm's prefix: they configure the tool, a step never sees them.
- Provisioners would move from `lib/pmm/provisioners/` to `lib/anfs/provisioners/`.

Not done now: a Containerfile is how container images are built, and a step that touches the
bootloader, the kernel or a running systemd has no meaning in one. What is done now is keeping
ssh out of the contract's required part, which costs nothing.

### 5. The build key

`build_key` replaces its `script <name> <sha>` lines with, per step in order:
`step <name> <provisioner>` and `sha256` of each path `fingerprint` printed (a directory is
walked, sorted). A provisioner whose `fingerprint` fails, fails the build.

### 6. Documentation

`pmm help` and CLAUDE.md gain the contract table. The `pmm-images` skill gains "writing a
provisioner". An ansible provisioner is not written here; the README names where it would go
(a stack package stowing `~/.local/lib/pmm/provisioners/yml`, declaring `brew: [ansible]`).

## Test Spec

- A fixture provisioner (`.fix`) in `$PMM_CONFIG_HOME/provisioners/`: its `run` is called with
  every variable in the table set, and `$ANFS_IMAGE_GUEST_EXEC id -u` prints 0; its exit status fails the build; its output is in the log.
- `fingerprint` naming a second file: touching that file changes the key; touching an unlisted
  one does not.
- A step with no provisioner fails `validate` naming the extension; a provisioner whose `check`
  prints something fails `validate` with that text.
- Steps of mixed provisioners run in name order.
- The `anfs` step in `host` mode: the key changes when a source gains a commit.
- tart (stubbed): `ANFS_IMAGE_GUEST_EXEC` uses the askpass helper, and the helper is gone after finalize.

## Verification

- [ ] `lib/pmm/build.sh` has no ssh call between "waiting for ssh" and finalize except through a provisioner
- [ ] `grep -r 'anfs:' machines/images/` finds nothing; `pmm image build anfs-dev` works
- [ ] `packages/pmm/tests/e2e debian`, and the macOS image (`ppm vm build macos`)
- [ ] `packages/anfs/tests/bash.bats` covers `provisioners/*` as entry points
