---
status: complete
started_at: "2026-10-13T07:20:00-04:00"
completed_at: "2026-10-13T08:50:00-04:00"
deviations: Library files renamed with their commands (image-test.sh, image-export.sh) and commands/completion/implode merged rather than prefixed. Image lookups from the machine side run in a subshell with the images' catalogue. run --rm removes at pmm down, not at a guest's own poweroff. Guest-side names and meta.yml fields lost "pim" too. PMM_SOCK_DIR and CLI_COMPLETE_ARGS added. Full list below.
summary: pim is folded into pmm on branch machine-images (not pushed, not installed). bats 301 of 301 and the real Debian e2e pass on macOS/hvf. Not verified - Linux/kvm, tart, e2e fedora, roundtrip, roundtrip-macos. The live debian-13 machine lost its socket files to an early test run and needs a down/up.
---

# Execution Log

`started_at` is the earliest trace of the work (the first bats run), not a recorded start.

## Read this first

1. **The live machine was harmed by a test run.** The first agent's `bats packages/pmm/tests`
   ran the implode tests as they were on main: `rm -rf /tmp/pmm-$(id -u)`, which is the socket
   directory of the real pmm. The running `debian-13` (qemu pid 43195) lost
   `/tmp/pmm-501/debian-13.qmp` and `debian-13.ga`. The VM is still up and ssh works. Until it is
   restarted, `pmm down debian-13` has only the ssh poweroff and then kill -9, and `pmm show`
   cannot ask the guest agent for an address. **Fix: `pmm down debian-13 && pmm up debian-13`**
   when convenient. Nothing touched it after that: `PMM_SOCK_DIR` was added, and the bats helper
   and the e2e set it. `/tmp/pmm-501` has kept its 07:20 mtime through every later run.
2. The same agent ran `pkill -f 'bats.*tests'` once. Any other bats run on the host at that
   moment would have died with it.
3. **Before installing this branch**: see "Before installing". Nothing is migrated.
4. The plan's Verification list is not complete: the two round trips have not run.

## What Was Done

- **Commands.** `pmm image list|show|build|test|export|remove|prune|path|cd|validate`, declared in
  `lib/pmm/commands.sh`. `image list` replaces `list` and `builds` (keys: `name`, `source`,
  `arch`, `state`, `key`, `disk`, `backend`, `user`, `sshkey`, `held`, `built`, `tested`).
  `verify` is `test` (`--test`, `test.sh`), `publish` is `export`, `clean` is `prune`. `hold`,
  `release`, `builds`, `migrate`, `create` and `up --image` do not exist.
- **`run`** (1a): `pmm run [image] [--name N] [--rm] [--no-start] [settings...]` makes a machine
  and starts it, named for the image, then `<image>-2`. It writes no definition: the settings are
  `settings:` in the machine's `state.yml`, read through `mach_q` (`settings.sh`) by `up`, `down`,
  `reset`, `shell`, `exec`, `show`, `list` and `remove`. A run machine's id is `-/<name>`.
- **The rest of 1b**: `exec`; the `snapshot list|create|remove` group; `remove` never deletes a
  definition; `network create` records in `$PMM_DATA_HOME/networks/<name>/state.yml` with no
  definition; `show` and `network show` end with the settings as definition YAML.
- **Libraries** (2): pim's files are in `packages/pmm/home/.local/lib/pmm/`. Settings are
  `PMM_IMAGE_*`; `PIM_INSTALL` is part of `PMM_INSTALL`. pmm sources nothing of pim's and runs
  no pim.
- **Layout** (2a): `machines/images/`, `machines/networks/` in core, the acme fixture and the
  local mirror (`~/.config/pmm/machines/{images,networks}`). `anfs_tool_dirs` is gone.
  `images`, `networks` and `image-...` are refused as machine names.
- **Data** (3): builds, keys, `exports/`, build workspaces and logs are under pmm's directories;
  tart builds are `pmm-image-<name>-<key>`. `_iso_adopt` is deleted.
- **Holds** (4): `holders_of` in `registry.sh` scans machines' and snapshots' `state.yml`.
  Holders read `web1` and `web1@s1`.
- **TCG warning** (5): `qemu_accel_warn`, once, from `image build`, `run` and `up`.
- **Everything naming pim** (6): `packages/pim/` deleted; `ANFS_TOOLS`, `bin/anfs`, `anfs.conf`,
  `packages/anfs/package.yml`, `ppm vm`, both round trips, comments, the acme README, CLAUDE.md
  (the pim section is an "Images" subsection of pmm's). Stack: ventoy, `docs/pdt.md`.
  `grep -rnw pim` finds one line in core (CLAUDE.md, history) and none in stack.
- **Tests** (7): pim's suites are `packages/pmm/tests/image-*.bats`, `qemu.bats`,
  `template.bats`; `holds.bats` is rewritten against the scan; one `e2e`, image half first.

## Commits

Core, `/Users/roberto/anfs-dev/machine-images`, `git log main..HEAD --oneline`
(103 files, +2257 -2081):

```
eda9787 pmm: run leaves no draft behind; image-... is not a machine name; snapshots complete
7dbc190 pmm: nothing names pim any more: anfs's package and settings, ppm vm, the round trips, CLAUDE.md
cdca6df pmm: run makes a machine from an image and writes no definition; exec, the snapshot group
3ae3c6c pmm: one directory in a source; images and networks are under machines/
fcce15f tests: bash.bats checks pmm's entry point where it named pim's
bbb88d6 pmm: images are pmm's (pmm image ...); pim's package is gone
```

Stack, `/Users/roberto/anfs-dev/machine-images-stack` (5 files, +5 -35):

```
88a1175 pim is pmm: ventoy's hint names anfs's ISO store, docs/pdt.md, firstmate's round trip
```

Neither is pushed. The verification pass (this log) changed no code and made no commit.

## Deviations from the Plan

Structure:

- **One catalogue per process.** The plan has `create` resolve the image into variables, then
  load machines. Instead the machine side asks about its image in a subshell that loads the
  images' catalogue (`image_def_dir`, `image_build`, `image_ensure_built` in `settings.sh`).
  `bin/pmm` loads images for `pmm image ...` and machines otherwise, after `cli_globals`;
  `__complete` loads none.
- **File names.** `verify.sh` is `image-test.sh` and `publish.sh` is `image-export.sh` (renamed
  with the commands; the plan kept `verify.sh`). `commands.sh`, `completion.sh` and `implode.sh`
  were merged into pmm's, not added as `image-` copies.
- **The build runs in-process**, so it is no longer called in an `||` list (that would switch
  errexit off inside it). `cmd_run` makes the machine in a subshell so the draft settings file
  goes whatever stops it.

Names the plan does not mention:

- Guest side: the default image user is `pmm`; `/etc/pmm-image`, `pmm-hostkeys.service`,
  `cloud.cfg.d/90_pmm.cfg`, `/pmm/authorized_keys` in the initrd, `/tmp/pmm-files`,
  `/tmp/pmm-askpass`. Build keys change (`pmm-image-build-v1`).
- `meta.yml`: `verified_key` is `tested_key`, `published` is `exports`.
- Left for plan 03, as the plan says: `PIM_IMAGE`, `PIM_ARCH`, `PIM_USER`, `PIM_FILES`, and also
  `PIM_SSH_PUBKEY`.
- Holder names have no `pmm:` prefix.

Behaviour:

- **`run --rm` removes the machine at `pmm down`**, or when a `--console`/`--display` run
  returns. pmm has no daemon: a guest that powers itself off stays until the next `pmm down`.
  `--rm` with `--no-start` is refused (64).
- A blank machine (`run --media M`, no image) requires `--name`.
- `run` has no `--network` (the reference lists none): only defined machines are on networks.
- `pmm up` with no machine is still a usage error, although declared `[machine...]`.
- `pmm network remove` does not delete a local definition either (the plan says it for machines).
- `network create` and `validate` refuse a subnet an installed, undefined network already has.
- `image list`: a build whose definition is gone has a null source and state `stale`. The tart
  boot user and password are in no list.
- `tart_rm` matches exactly `pmm-image-<name>-(<key>|building|test)`: removing `foo` no longer
  deletes `foo-bar`'s builds.
- `pmm install src/` dies when the source has no `machines/`.
- `snapshot create` with no name uses the time; `snapshot remove` asks unless `-y`;
  `snapshot list` is an `anfs_table` (`name`, `build`), so piped it is two columns.
- `pmm show` prints the YAML block for defined machines too.
- The `image-...` name rule covers every backend, so `pmm run` of an image itself named
  `image-...` needs `--name`.

Additions:

- `PMM_SOCK_DIR` (see "Read this first").
- `CLI_COMPLETE_ARGS` in `lib/anfs/cli.sh`: a completion function gets the arguments already on
  the line. `complete_snapshot` uses it for `reset` and `snapshot remove` (qemu machines only).
- Options beyond the reference, all supersets: `image path --disk --arch`, `image prune --isos`
  (nothing else can delete an ISO until plan 04), `--arch` on `export` and `test`,
  `shell/exec -u`.
- `network create` usage shows `[--kind KIND]`: `cli.sh` cannot declare a required option; it is
  enforced at run time (64).

Outside the plan's list:

- Stack `packages/packer/home/.config/zsh/packer.zsh`: `pim-packer-init` and
  `PIM_PACKER_CONFIG_HOME` deleted. **Revert that hunk of 88a1175 if the helper is still used.**
- Stack `packages/firstmate/tests/roundtrip`: `PIM_INSTALL` is `PMM_INSTALL`.
- `ppm vm snapshot` calls `pmm snapshot create|list`.
- roundtrip's "the commands that went to pmm say so" check is deleted, nothing in its place.
- The pre-commit hook bumped `packages/qemu/package.yml` to 0.1.1.

## Test Results

Final pass, on eda9787 with a clean tree, macOS 26.6.2 arm64, qemu 11.1.1, bats-core.

bats, each as `bats packages/<tool>/tests </dev/null`:

| Suite | Result | Time |
| --- | --- | --- |
| pmm | 79 of 79 | 203 s |
| anfs | 127 of 127 | 31 s |
| ppm | 28 of 28 | 4 s |
| pcm | 67 of 67 | 35 s |

301 tests, 0 failures. psm and wsm were not run (not touched).

`packages/pmm/tests/e2e debian </dev/null`: real qemu on hvf, isolated HOME, the shared ISO store.
Passed on the first run of this pass (no fix round needed), exit 0, 434 s in all. Earlier agents report three passes on the branch
(after bbb88d6, cdca6df and eda9787). Verbatim:

```
== debian-13: the image
  ok    image validate (0s)
  ok    image list: unbuilt (0s)
  ok    image build --test (188s)
  ok    rebuild is a cache hit (1s)
  ok    image list: built and tested (0s)
  ok    image export -c, a standalone qcow2 (48s)
== debian-13: machines
  ok    run --no-start: a, with a read-only share (2s)
  ok    no definition was written (0s)
  ok    the build is held by a (0s)
  ok    up a (25s)
  ok    exec runs a command (1s)
  ok    the share is read-only (0s)
  ok    the profile's one cpu (0s)
  ok    run: a second machine, started (28s)
  ok    ps lists both (0s)
  ok    b has its own disk (0s)
  ok    down a (1s)
  ok    up again: the disk and the settings persisted (26s)
  ok    snapshot, change, reset to it (54s)
  ok    the snapshot holds the build too (0s)
  ok    image remove is refused while held (0s)
  ok    reset onto the build: a new disk (28s)
  ok    list: a and b running (0s)
  ok    snapshot remove (0s)
  ok    run --rm: gone at its shutdown (29s)
  ok    rm a and b (3s)
  ok    the build is no longer held (0s)
  ok    image prune (0s)
  ok    image remove, now that nothing is on it (0s)
  ok    implode (0s)
  ok    implode left nothing
== PASS
```

Timings worth knowing: build plus test 188 s; an export with `-c` 48 s; a boot to ssh 25 to 29 s.

Afterwards: the only qemu running is the user's `debian-13` (pid 43195), `/tmp/pmm-501` is
unchanged, no `/tmp/pmm-e2e.*` or `/tmp/pmm-bats-501` content is left, both worktrees are clean.

## Not Verified on This Machine

- **Linux and kvm**: no build or machine has run there. The TCG warning, `PMM_KVM_DEV`, and all
  of networks (libvirt, the bridge helper, interfaces, `network create` with no definition) are
  covered by bats with `virsh` stubbed, nothing more.
- **tart**: the `pmm-image-<name>-<key>` naming, `tart_rm`'s pattern, tart machines and their
  snapshots, the two-guest limit. Stubbed in bats; the e2e points `TART_HOME` at nothing.
  `ppm vm build macos` (now `pmm image build`) has not run.
- **`e2e fedora`**: not run, by instruction.
- **`packages/anfs/tests/roundtrip debian`** and **`roundtrip-macos`**: edited, `bash -n` only.
  Their new checks (`pmm image list --json`, `pmm image validate`, `pmm image show`) passed by
  hand against the acme fixture under an isolated HOME, not in a box. These are in the plan's
  Verification list, so that list is open.
- Stack's `packages/firstmate/tests/roundtrip`: `bash -n` only.
- **A real install**: `ppm install core/anfs` from this branch, `command -v pim` finding
  nothing, `~/.local/lib/pim` not existing, `pmm implode -y` against real XDG homes and tart.
- TCG (a foreign-arch guest) for real.
- psm and wsm bats suites.

## Before Installing This Branch

Nothing is migrated and nothing forwards. With the **old** tools still installed, in this order:

1. `pmm down` every machine and `pmm remove` them (or `pmm implode -y`): old machines are
   overlays on pim's builds and their records name pim's keys and paths. The running `debian-13`
   is one of them; its sockets are gone, so `down` will take the ssh-poweroff path.
2. `pim implode -y`.
3. Delete tart's leftovers: `tart list`, then `tart delete` each `pim-*` VM (and any `pmm-*`
   left from step 1).
4. Move local definitions by hand if there are any: `~/.config/pim/images/` to
   `~/.config/pmm/machines/images/`, `~/.config/pmm/networks/` to `~/.config/pmm/machines/networks/`.
   A local `machine.yml` that `create` or `up --image` wrote is now a definition like any other:
   keep it or delete it.
5. In any image of your own: `verify.sh` is `test.sh`. In any other source: `images/` and
   `networks/` go under `machines/`.
6. `PIM_*` in `anfs.local.conf` or a `user/anfs` `anfs.conf`: `PMM_IMAGE_INSTALL_TIMEOUT`,
   `PMM_IMAGE_SSH_TIMEOUT`, `PMM_INSTALL`.
7. `ppm remove core/pim` while its package directory still exists (it is gone from the source
   on this branch; I did not check what ppm does with a tracker whose package has vanished).
   Then merge and install. Do not use `ppm install -r core/anfs`.
8. Every image rebuilds (new build keys); the ISO store is kept.

## Left Open

- `roundtrip debian`, `roundtrip-macos` (above).
- `state.yml` still carries the unread `base` field (predates the branch).
- A tart machine named `x-snap-y` could clash with machine `x`'s snapshot `y` (predates the branch).
- Snapshot-name completion is qemu only.
- No `pmm network cd` (the reference lists none).
- Plan 02: `image show` printing what a build runs; `image new`.
