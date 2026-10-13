---
---

# Plan 02 — The image directory is complete

## Context — read these files first

- `lib/pmm/image.sh` (was pim's) — `img_answer_file`, `img_scripts`, `img_verify_script`, `tpl_load`
- `lib/pmm/build.sh` — `build_key`, `_build_qemu`
- `lib/pmm/defaults/` — what moves out
- `machines/images/debian-13/`, `machines/images/fedora-44/`, `machines/images/anfs-dev/`, `machines/images/anfs-test-macos/`
- `packages/anfs/home/.local/lib/anfs/iso.sh` and stack's `isos/isos.yml` — the catalogue
- A pcm service directory (`stack/containers/gemstash/`) — the shape being matched
- The `pcm-containers` skill — the model for the skill this plan adds

## Overview

Someone who opens `machines/images/debian-13/` today finds one file, and has to know that the preseed, the
base script and the verify script are in pim's library, and that the answer file's variables get
their defaults from a bash function. After this plan the directory holds all of it, as a pcm
service directory does.

```
machines/images/<name>/
  image.yml          what it is: backend, distro, arch, iso | from | base, user, vars, machine
  preseed.cfg | kickstart.ks     root images: required
  provision/         steps, run in name order (plan 03 makes the extension choose the provisioner)
  files/             copied to the guest for the steps
  test.sh            what `pmm image test` runs
  README.md          what the image is for and what it expects
```

## Implementation

### 1. Defaults move into the images

- `defaults/debian/{preseed.cfg,scripts/10-base.sh,verify.sh}` (as `test.sh`) go to `machines/images/debian-13/`;
  `defaults/fedora/*` to `machines/images/fedora-44/`. `defaults/` keeps `finalize.sh` only, which stays
  pmm's: it is what makes any build a clean image, and an image must not be able to skip it.
- `scripts/` is renamed `provision/`. `scripts: {defaults: ...}` is removed from `image.yml`.
- A root image with no answer file fails `validate`. A child (`from:`) has none.
- `test.sh` is the image's own, else the nearest one up its `from:` chain. With none,
  `image test` says so and succeeds, as now.
- `img_scripts` becomes: the image's `provision/*` in name order. A child does not rerun its
  parent's steps (it never did).
- The build key is computed from the same inputs; the paths change, so every key changes once.

A new root image for a distro that already has one copies that image's files. `image new`
(step 5) does it.

### 2. Variables are all in `image.yml`

`tpl_load` sets thirteen defaults in bash (`LOCALE`, `KEYBOARD`, `TIMEZONE`, `DOMAIN`,
`MIRROR_HOST`, `MIRROR_PATH`, `HTTP_PROXY`, `PARTITIONING_*`, `TASKSEL`, `PACKAGES`,
`GRUB_DEVICE`). They move to `vars:` in the shipped root images. `tpl_load` keeps only what an
image cannot know: `USER`, `FULLNAME`, `PASSWORD_HASH`, `KS_PASSWORD`, `SSH_PUBKEY`, `IMAGE`,
`ARCH`, and `HOSTNAME` defaulting to the image's name. An unknown `${VAR}` is already an error,
so an answer file that names a variable its `image.yml` does not set fails at build, not
silently. The parallel arrays in `tpl_set`/`tpl_get` become one associative array while this is
open (the bash floor).

### 3. ISOs by catalogue id

```yaml
iso:
  arm64: debian-13-arm64
  amd64: debian-13-amd64
```

- Core gets `isos/isos.yml` with the entries its images name (stack's catalogue is not below core
  in priority, so core cannot depend on it).
- An entry an image names must carry `checksum: sha256:<hex>`: the hash is part of the build key.
  `validate` refuses an entry with only `checksum_url`.
- Resolution is the catalogue's own (`~/.config/anfs/isos.yml`, then the sources in order), so a
  user's catalogue can repoint an id at a mirror. The hash in the key keeps that honest.
- `iso.<arch>.url` and `.sha256` inline are removed, with `iso_fetch`'s own URL loop.

### 4. `show` prints the plan

`pmm image show <image>` adds, before the builds:

```
  answer      ~/.local/share/anfs/sources/core/machines/images/debian-13/preseed.cfg
  vars        HOSTNAME=debian LOCALE=en_US.UTF-8 ...
  iso arm64   debian-13-arm64  (~/.cache/anfs/isos/debian-13.7.0-arm64-netinst.iso, present)
  provision   10-base.sh       core/machines/images/debian-13/provision/10-base.sh
              finalize         (pmm)
  test        ~/.local/share/anfs/sources/core/machines/images/debian-13/test.sh
  logs        ~/.local/state/pmm/logs/debian-13-arm64/
```

For a child, the chain's steps are listed under each image they come from. `--json` gives the
same as an object (`anfs_result`).

### 5. `image new`

`pmm image new <name> [--from <image>] [--copy <image>] [--source <alias>]`

- `--from debian-13`: a child. Writes `image.yml` (`from:`), an empty `provision/`, a README stub.
- `--copy debian-13`: a root. Copies that image's directory under the new name.
- Neither: asks for one; without a terminal, exit 64.
- Lands in `~/.config/pmm/machines/images/` unless `--source` names a source. It only creates, so it
  never asks; an existing directory is refused with 73.

### 6. Fixes found in the audit

- The help for `export` said the default output is `./<image>-<arch>.qcow2`; it is
  `$PMM_DATA_HOME/exports/<name>-<arch>-<key>.qcow2`. The declaration is corrected.
- `image.yml`'s `cpus`, `memory`, `disk` move under `build:` (`build: {cpus, memory, disk}`), so
  the file reads as two blocks: what the build runs with, and `machine:` for what machines get.
  `disk` under `build:` is still the image's disk size.

### 7. README and skill

- `machines/images/debian-13/README.md`, `fedora-44`, `anfs-dev`, `anfs-test-macos`: what it is, its user,
  what a child inherits.
- A `pmm-images` skill in `skills/`, on the pattern of `pcm-containers`: creating an image,
  adding a provision step, overriding a shipped image from a higher source (`from:` its own name),
  making `validate` pass.

## Test Spec

- `validate`: a root with no answer file fails; an iso id not in the catalogue fails; an id whose
  entry has no pinned sha256 fails; inline `iso.<arch>.url` fails with what replaced it.
- `build_key` changes when: the answer file, a `vars:` value, a provision step, a file under
  `files/`, the catalogue entry's hash changes. It does not change for `description`, `machine:`
  or `README.md`.
- `show` lists the answer file, each step and the test script with paths; for a child, both images' steps.
- `new --from`, `new --copy`, an existing name (73), no terminal and neither option (64).
- A user-source `machines/images/debian-13/` with `from: debian-13` and one step builds on core's.

## Verification

- [ ] `lib/pmm/defaults/` contains `finalize.sh` and nothing else
- [ ] `pmm image show debian-13` names every file the build reads
- [ ] `pmm image build --test debian-13` and `fedora-44` pass (e2e)
- [ ] `pmm image new web --from debian-13 && pmm image build web`
- [ ] `psm skills list` shows `pmm-images`
