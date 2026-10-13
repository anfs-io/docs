---
status: complete
started_at: "2026-10-13T09:50:00-04:00"
completed_at: "2026-10-13T11:22:00-04:00"
deviations: A shadowed definition has its own build record (meta@<source>.yml) and a build that is there is adopted, never rebuilt. image build -f refuses while something is on the disk. cli_call sets CLI_COMMAND. validate refuses the old shape and checks build:. The build key leaves out timeouts, the arch list and the iso ids. image show's format differs from the plan's example. Stack has no commit. Full list below.
summary: The image directory is complete on branch machine-images of core (four commits, not pushed, not installed). bats anfs 142, pmm 97, ppm 28, pcm 67, psm 8, wsm 56, stack ventoy 18, all passing; the real e2e passes for debian and for fedora on macOS/hvf; a same-name override was built for real on core's debian-13. Not verified - tart, Linux/kvm, amd64, the round trips, a real anfs-dev build, the interactive image new prompt, the real skills CLI.
---

# Execution Log

`started_at` is the end of plan 04's verification, not a recorded start. The verification pass
(this log) changed no code and made no commit.

## Read this first

1. **Every image rebuilds once more.** The paths, the `vars:` and the shape of `image.yml`
   changed, and the fix round changed what the key hashes. Nothing is installed from this branch
   yet, so nothing is lost.
2. **An image of your own in the old shape does not validate.** `scripts/` is `provision/`;
   `scripts:` is gone; `cpus`, `memory`, `disk` are under `build:`; `iso.<arch>` is a catalogue
   id; a root image needs its own answer file and its own `vars:`. `validate` names each one and
   what replaced it. `pmm image new <name> --copy debian-13` is the quick way to a root that works.
3. **The provision environment is still `PIM_IMAGE`, `PIM_ARCH`, `PIM_USER`, `PIM_FILES`,
   `PIM_SSH_PUBKEY`.** Plan 03 renames them; the READMEs, `pmm help` and the `pmm-images` skill
   name them as they are and go with it.
4. Plan 01's "Before installing this branch" still holds, with item 5 extended by 2 above.

## What Was Done

- **Defaults moved into the images** (1). `lib/pmm/defaults/` holds `finalize.sh` and nothing
  else. The preseed, the kickstart, the base steps and the tests are in
  `machines/images/debian-13/` and `fedora-44/`. `scripts/` is `provision/` (also in
  `anfs-test-macos` and the acme fixture). A root image with no answer file fails `validate`.
  `test.sh` is the image's own, else the nearest one up its `from:` chain.
- **Variables** (2). The thirteen defaults are `vars:` in the two root images. `tpl_load` keeps
  `USER`, `FULLNAME`, `PASSWORD_HASH`, `KS_PASSWORD`, `SSH_PUBKEY`, `IMAGE`, `ARCH` and
  `HOSTNAME`. The template variables are one associative array (`TPL`). A multi-line value is one
  variable.
- **ISOs by catalogue id** (3). `iso.<arch>` is an id in the catalogue (core's `isos/isos.yml`,
  which plan 04 brought forward), whose entry must pin a sha256. `validate` refuses an unknown
  id, an entry with only `checksum_url`, and the inline map. `iso_fetch`'s own URL loop is gone;
  `packages/pmm/tests/iso.bats` is deleted.
- **`image show` prints the plan** (4): answer file, `vars:`, each arch's ISO with its file and
  whether it is there, every step with its path (a child's parents first), the anfs install,
  finalize, the test and the log directory, then the builds. `--json` is the same as one object.
  Both come from `_image_plan` in `registry.sh`.
- **`image new <name> --from|--copy [--source]`** (5): 73 for an existing directory, 64 with
  neither option and no terminal, 64 with both.
- **Audit fixes** (6): `export`'s help names the real default; `cpus`, `memory`, `disk` are under
  `build:`.
- **README and skill** (7): a README in each of the four shipped images;
  `skills/pmm-images/SKILL.md`.
- **Also**: `pmm help`, CLAUDE.md, README.md, the bats helper (test ISOs are in a local
  catalogue), the e2e (the repo is the source `core`; a child made with `image new` is built,
  tested and run).

## Commits

Core, `/Users/roberto/anfs-dev/machine-images`, `git log 616bc88..HEAD --oneline`
(45 files, +1572 -336):

```
2234bcc pmm: a build that is there is never built again; a shadowed definition's machines hold its build
2471df2 pmm: image show lists no finalize for a tart image (its build ends with a reboot)
ec72bc3 pmm: the image directory is complete; ISOs by catalogue id; image show prints the plan; image new
bfb555f cli: cli_call makes the command it runs the running one, so its usage errors word its own line
```

Stack, `/Users/roberto/anfs-dev/machine-images-stack`: no commit, HEAD is still 4ffc4a3. It has
no pmm images and nothing that names the old shape.

Not pushed. 2234bcc is the fix round after review (below).

## Deviations from the Plan

Beyond the plan, and the largest:

- **A shadowed definition has its own build record**, `meta@<source>.yml` beside `meta.yml`
  (`img_bname` in `registry.sh`). The Test Spec's "a user-source `debian-13` with
  `from: debian-13` builds on core's" did not really work with one `meta.yml` per name: each
  build reinstalled the root over the disk the child was an overlay on.
- **A build that is there is never built again** (`build_one`, `build_adopt`). If the disk (or
  tart VM) of the wanted key exists, it is taken as built whatever the records say, and the
  record moves between `meta.yml` and `meta@<source>.yml` with `built_at`, `tested_key` and
  `exports`. This is what makes adding or removing an override safe; it also covers a definition
  that moved source or went back to an earlier key. With no record naming the key a fresh one is
  written, so `built_at` is then the adoption time.
- **`image build -f` refuses (exit 1)** while a machine, a snapshot or another image's build is
  on the disk it would replace. A behaviour change to `-f`.
- A machine from a shadowed definition records the plain image id (`core/debian-13`), so it
  holds the build.
- **`lib/anfs/cli.sh`**: `cli_call` sets `CLI_COMMAND` while the command it runs is running, so
  `cli_usage_err` from it words its own usage line. Found by the `image new` 64 test.

`validate`:

- Errors for the old shape, each naming what replaced it: a `scripts/` directory, a `scripts:`
  key, top-level `cpus`/`memory`/`disk`. The plan asked this only for the inline ISO url.
- `build:` is checked: a map of `cpus`, `memory`, `disk` only; `cpus` and `memory` positive
  integers, `disk` `<n>G` or `<n>M` (tart: `<n>G`). `build.memory: 4G` is refused.
- Warnings for an answer file, `vars:`, `user:`, `distro`, `iso`, `base` or `arch` on a `from:`
  image (by presence of the key), and for anything in `provision/` that is not `*.sh`.

The build key:

- `timeouts`, the `arch` list and the `iso` ids are left out of the hashed `image.yml`; this
  arch's pinned sum stays in. `build.cpus` and `build.memory` stay in, on purpose.
- `PMM_IMAGE_BUILD_VERSION` was not bumped: every key changes anyway.
- `meta.yml`'s `iso` field records the catalogue id where it recorded the first URL.

`image show`:

- Not the plan's example: `vars` are one per line (values hold spaces and commas), steps show
  full `~` paths, a child's steps are one list in build order with each path naming its image
  (no per-image headers), the anfs install is listed as a pmm step, labels pad to the longest.
- A tart image shows `base` and no answer, ISO or finalize (a tart build never ran finalize).
- `--json` is assembled by hand with `table.sh`'s `_anfs_json_str`, not `anfs_result`, which is
  flat.
- A multi-line `vars:` value shows on one line with `\n`; `--json` is exact.

`image new`:

- `--from <source>/<name>` writes the qualified `from:`. A plain name that resolves to a
  different image from where the new one lands gets a warning naming both.
- `--from` writes `provision/.gitkeep`. After creating, it validates and warns rather than
  refusing. `--copy` copies the README and `description:` verbatim and says so.
- The catalogue check takes a checksum with or without the `sha256:` prefix.
- `complete_source` was added for `--source`.

Other:

- Only `provision/*.sh` run, as before. Dispatch by extension is plan 03's.
- `img_vars` emits `NAME=<base64>` lines (`var_value` decodes).
- tart: logs go to `logs/<bname>-<arch>`; an image's VMs are matched by name as text.
- The e2e no longer points `PMM_IMAGES_HOME` at the repo: the repo is registered as the source
  `core` in the temp HOME. It gained seven steps.
- Three core commits for the plan and one for the review's findings, not one per step.

## Review and Fixes

The review found the plan's steps sound and two defects in the shadowed-build record, both fixed
in 2234bcc with bats tests: adding or removing an override reinstalled the root over a disk that
machines and children were on; a machine from a shadowed definition recorded
`core/debian-13@core` and held nothing. The minor ones went with them: `image new --json`,
`--from` dropping the source, `build:` and a child's `vars:`/`user:` unvalidated, multi-line
`vars:` values split.

Until this pass those fixes had run against the stubs only. They have now run for real (below).

## Test Results

Final pass, on 2234bcc with a clean tree in both worktrees, macOS 26.6.2 arm64, qemu 11.1.1,
bats-core. No failure was found and no code was changed.

bats, each as `bats packages/<tool>/tests </dev/null`:

| Suite | Result | Time |
| --- | --- | --- |
| anfs | 142 of 142 | 46 s |
| pmm | 97 of 97 | 307 s |
| ppm | 28 of 28 | 4 s |
| pcm | 67 of 67 | 36 s |
| psm + wsm | 64 of 64 (8 + 56) | |
| stack ventoy (`ANFS_TEST_LIB` set to the core worktree's lib) | 18 of 18 | |

416 tests, 0 failures, none skipped.

`packages/pmm/tests/e2e debian </dev/null`: real qemu on hvf, isolated HOME, the shared ISO
store (read only: the ISO was there). Passed on the first run, exit 0, 522 s. Verbatim:

```
== debian-13: the image
  ok    image validate (1s)
  ok    image show: every file the build reads, and its ISO (0s)
  ok    image list: unbuilt (0s)
  ok    image build --test (186s)
  ok    rebuild is a cache hit (1s)
  ok    image list: built and tested (0s)
  ok    image export -c, a standalone qcow2 (46s)
  ok    image new --from: a child, in the local source (1s)
  ok    the child builds on debian-13's build and passes its test.sh (59s)
  ok    image show web: both images' steps; built and tested (1s)
  ok    a machine from the child has its step's work (29s)
  ok    image remove is refused while the child is built on it (0s)
  ok    image remove web (0s)
== debian-13: machines
  ok    run --no-start: a, with a read-only share (2s)
  ok    no definition was written (0s)
  ok    the build is held by a (1s)
  ok    up a (25s)
  ok    exec runs a command (0s)
  ok    the share is read-only (0s)
  ok    the profile's one cpu (0s)
  ok    run: a second machine, started (28s)
  ok    ps lists both (0s)
  ok    b has its own disk (1s)
  ok    down a (1s)
  ok    up again: the disk and the settings persisted (25s)
  ok    snapshot, change, reset to it (54s)
  ok    the snapshot holds the build too (0s)
  ok    image remove is refused while held (1s)
  ok    reset onto the build: a new disk (27s)
  ok    list: a and b running (1s)
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

`packages/pmm/tests/e2e fedora </dev/null`: the same 38 steps, exit 0, 519 s, first run.
`ANFS_ISO_HOME` was a temp store, so the real one was not added to: `pmm iso pull
fedora-44-arm64` fetched the 1.2 GB ISO from dl.fedoraproject.org into it and checked the
catalogue's sha256. `image build --test` 253 s, export 57 s, the child's build and test 43 s, a
boot to ssh 17 to 21 s.

## Verification

The plan's list, under isolated homes (the worktree as the source `core`):

- [x] `lib/pmm/defaults/` contains `finalize.sh` and nothing else.
- [x] `pmm image show debian-13` names every file the build reads: the preseed, the thirteen
  `vars:`, both ISOs with their store path and `missing`/present, `provision/10-base.sh`,
  finalize with its path, `test.sh`, the log directory. Also read for `anfs-dev` (its parent's
  step, `anfs install (host)`, finalize), `fedora-44` and `anfs-test-macos` (`base`, one step,
  the test; no answer, ISO or finalize). `image validate`: 4 images, 0 errors, 0 warnings.
  `show --json` parses.
- [x] `pmm image build --test debian-13` and `fedora-44` pass (both e2e runs).
- [x] `pmm image new web --from debian-13 && pmm image build web` (the e2e, both distros: the
  child reuses the parent's build, runs only its own step, passes the parent's `test.sh`, and a
  machine from it has the step's work). By hand: an existing name is 73, neither option without
  a terminal is 64, both options is 64, `--copy` gives a root that validates, `--from
  core/debian-13` writes the qualified `from:`, `--json` is one object.
- [x] `psm skills list` shows `pmm-images` (`core`, `missing`), with npx stubbed as psm's bats
  does. Not with the real skills CLI.

The Test Spec's last line and the review's two findings, for real (qemu on hvf, an isolated home,
the Debian ISO read from the real store):

1. `core/debian-13` built (158 s), machine `m` run from it, a file written, `m` stopped.
2. A source `user` added above core with `machines/images/debian-13/` (`from: debian-13`, one
   step). `image list` shows `user` as `stale` on core's key and core's as `shadowed`.
3. `image build --test debian-13`: "core/debian-13 (arm64) is up to date", then only
   `50-mine.sh` and finalize on an overlay (29 s), and core's `test.sh` passes. **The root disk's
   inode, mtime and size are the same before and after.** `meta.yml` is the override's,
   `meta@core.yml` core's.
4. Machine `o` from the override has the step's file. `m` boots again with its own file and
   without the override's: its backing disk is intact. `pmm show` gives `core/debian-13` for `m`
   and `user/debian-13` for `o`.
5. `image remove -y core/debian-13` is refused (1, "has machines on it: m o").
   `image build -f core/debian-13` is refused (1).
6. The override removed again: `image build debian-13` says up to date, the root disk is still
   the same inode, both machines still answer. After `rm m o`, `image prune` deletes the
   override's disk; implode leaves nothing.

Afterwards: no qemu is running, `/tmp/pmm-501` has kept its 07:20 mtime, the real ISO store is
byte for byte as it was (`ls -la` diffed), no `/tmp/pmm-e2e.*` is left, both worktrees are clean.

## Not Verified on This Machine

- **tart**: no build. `anfs-test-macos` in the new shape validates and its plan prints, nothing
  more. `tart_build`'s keeping of a same-name parent's VM, tart adoption and the tart log
  directory have no test at all; `_tart_image_vms`/`tart_rm` have a stubbed one. A tart machine
  was running on this host and was left alone.
- **Linux and kvm**: nothing ran there.
- **amd64**: both e2e runs are arm64. The amd64 ISO entries (urls, sums) were not fetched, and no
  amd64 guest was installed (TCG).
- **`anfs-dev`**: not built. Validate, `image show` and the bats chain checks only, so the
  `anfs:` install step did not run in a guest.
- **`packages/anfs/tests/roundtrip`** and **`roundtrip-macos`**: not run. The acme fixture's
  rename to `provision/` is covered by bats only, not in a box.
- **The interactive `image new` prompt** (neither option, with a terminal): never exercised.
- **`psm skills list` with the real skills CLI**; the `pmm-images` skill has no bats test, and
  nobody has used it to make an image.
- **A real install** of this branch (`ppm install core/anfs`), and anything against real XDG
  homes.
- The Debian ISO was not downloaded in this pass (it was in the store); Fedora's arm64 one was.
- `image build -f` was only seen refusing, for real. A forced rebuild of a root with nothing on
  it ran against the stubs only.
- A user catalogue (`~/.config/anfs/isos.yml`) repointing an id at a mirror: bats only.
- Adoption with no record at all (a disk of the right key and no `meta*.yml`): bats only.

## Left Open

- `image list` shows a new override as `stale` on the lower definition's key until the override
  is built (seen in the real run, step 2). True, if surprising.
- `image remove <source>/<name>` refuses for the holders of every build under the name, not only
  that definition's (step 5 names `o`, which is on the override's build).
- A machine whose image is a now-shadowed definition not rebuilt since the override was added
  gets no "newer build" hint in `pmm show`.
- `build_key` yields a key for an unknown ISO id; the build then dies in `iso_fetch`.
- `image new --copy` leaves the original's README and `description:`.
- `image show` prints a path outside `$HOME` in full (the store and logs in a temp home); only
  cosmetic.
- By hand, a `PMM_SOCK_DIR` deep in a temp directory fails the build with qemu's "UNIX socket
  path is too long" (104 bytes on macOS) in `install.log`; pmm reports only "the installer
  failed". Mine to avoid, not a regression, but pmm could say so.
- From plan 01, unchanged: the two round trips, `state.yml`'s unread `base`, the tart snapshot
  name clash.
- Plan 03: the `PIM_*` rename, dispatch by extension in `provision/`.
