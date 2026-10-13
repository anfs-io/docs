---
status: complete
started_at: "2026-10-13T11:22:00-04:00"
completed_at: "2026-10-13T13:40:00-04:00"
deviations: ANFS_IMAGE_GUEST_EXEC and _PUSH are paths to scripts pmm writes, not ssh command lines. ANFS_IMAGE_WORK added to the contract. Every regular file in provision/ is a step. The anfs step's installer is a key of the step (PMM_IMAGE_ANFS_INSTALLER_URL is gone). validate runs each step's fingerprint, refuses a leftover anfs: key and requires an account name for user.name. A fingerprint must name the step first. The provisioner's own file is not in the build key. Stack has no commit. Full list below.
summary: A provision step is run by a provisioner on branch machine-images of core (three commits, not pushed, not installed); sh and anfs ship behind the contract, and nothing names a PIM_ variable. bats anfs 142, pmm 112, ppm 28, pcm 67, psm 8, wsm 56, stack ventoy 18, all passing; the real Debian e2e passes on macOS/hvf; anfs-dev and a root image with user.authorized_keys were built and run for real. Not verified - tart for real (ppm vm build macos), Linux/kvm, amd64, e2e fedora, the anfs step's pushed mode, the round trips, a real install.
---

# Execution Log

`started_at` is the end of plan 02's verification, not a recorded start. The verification pass
(this log) changed no code and made no commit.

## Read this first

1. **Every image rebuilds once more.** The key's `script` lines are `step` lines, and the steps
   see other variables. Nothing is installed from this branch yet, so nothing is lost.
2. **A step of your own that reads `PIM_IMAGE`, `PIM_ARCH`, `PIM_USER`, `PIM_FILES` or
   `PIM_SSH_PUBKEY` gets nothing.** They are `ANFS_IMAGE_NAME`, `_ARCH`, `_USER`, `_FILES` and
   `_SSH_PUBKEY`. Nothing warns: the old names are simply unset.
3. **An `anfs:` key in an `image.yml` of your own does not validate.** It is a step,
   `provision/50-toolkit.anfs`, with the same keys. `PMM_IMAGE_ANFS_INSTALLER_URL` in an
   `anfs.conf` is no longer read: the step's `installer:` key replaced it.
4. **Every regular file in `provision/` is now a step**, and one nothing provides is a validate
   error. Plan 02 only warned about a file that was not `*.sh`. A dotfile or a directory there is
   still not a step.
5. `files/` arrives in the guest owned by root (it was the ssh user's): `GUEST_PUSH` copies as
   root with `--no-same-owner`.
6. The plan's Verification list is not complete: `ppm vm build macos` did not run (tart, below).
7. Plans 01 and 02's "Before installing this branch" still hold, with 2 to 4 above added.

## What Was Done

- **The seam** (1; `lib/pmm/provision.sh`, new). The provisioner of `provision/<NN>-<name>.<ext>`
  is `<ext>`: the first executable of `$PMM_CONFIG_HOME/provisioners/<ext>` and
  `$PMM_LIB_DIR/provisioners/<ext>`, called with `check`, `fingerprint <step>` or `run <step>`,
  in the image's directory.
- **What `run` gets** (2). `guest_ssh` in `ssh.sh` writes two scripts into the build workspace,
  `exec` and `push`, handed over as `ANFS_IMAGE_GUEST_EXEC` and `ANFS_IMAGE_GUEST_PUSH`. `exec`
  quotes each argument for the shell at the far end of ssh, so a provisioner passes argv.
  `run_script` and `push_dir` are gone. With them go `ANFS_IMAGE_GUEST_TRANSPORT` (`ssh`),
  `_HOST`, `_PORT`, `_USER`, `_KEY`, and `ANFS_IMAGE_NAME`, `_ARCH`, `_BACKEND`, `_DISTRO`,
  `_USER`, `_DIR`, `_FILES`, `_SSH_PUBKEY`, `_WORK`.
- **tart**. The askpass helper goes in once before the first step (`_tart_guest`) and comes out,
  with `/tmp/pmm-files`, after the last one and before the reboot (`_tart_guest_close`). tart
  builds push `files/` too, which they never did.
- **`sh`** (3; `lib/pmm/provisioners/sh`). The script is piped to
  `$ANFS_IMAGE_GUEST_EXEC env ... bash -s`. finalize and `test.sh` are run by it like any `.sh`
  step.
- **`anfs`** (4; `lib/pmm/provisioners/anfs`). `lib/pmm/anfs.sh` is deleted and the `anfs:` key
  is gone from `image.yml`. `anfs-dev` has `provision/50-toolkit.anfs` (`sources: host`).
- **The build key** (5). Per step, `step <name> <provisioner>` and the sha256 of what
  `fingerprint` names (`key_step`, which `validate` reads a step through as well). `build_key`
  returns 2 for a step nothing provides or a fingerprint that fails.
- **`validate`**: a step nothing provides, a file with no extension, anything `check` prints, a
  failing fingerprint, and a leftover `anfs:` key are errors.
- **Shipped images**: `debian-13/test.sh` and `anfs-test-macos/provision/10-provision.sh` read
  `ANFS_IMAGE_*`. `fedora-44` needed nothing (it reads no contract variable).
- **Docs** (6): `pmm help` (the contract, both tables), CLAUDE.md (a "Provisioners" subsection),
  the `pmm-images` skill ("Writing a provisioner"), the `anfs-dev` README, `image new`'s README
  template.
- **Tests**: `packages/pmm/tests/provision.bats` (14 tests) covers every line of the Test Spec;
  `bash.bats` lists `provisioners/*` as entry points; the e2e builds its child image with a `.sh`
  step and a provisioner of the run's own (`.probe`).
- **Nothing names a `PIM_` variable** in either worktree; `provision.bats` holds that for core.

## Commits

Core, `/Users/roberto/anfs-dev/machine-images`, `git log 2234bcc..HEAD --oneline`
(28 files, +1288 -237):

```
960b5bb pmm: validate reads a step's fingerprint as the build key does; the anfs step quotes packages and refuses unknown keys
e41c734 pmm: the anfs step's install finds the user's systemd session
0d947e5 pmm: a provision step is run by a provisioner; sh and anfs behind the contract
```

Stack, `/Users/roberto/anfs-dev/machine-images-stack`: no commit, HEAD is still 4ffc4a3. It has
no pmm image, no provisioner and nothing that named a `PIM_` variable.

Not pushed. 0d947e5 is the plan; e41c734 a fix found by the first real `anfs-dev` build; 960b5bb
the fix round after review (below).

## Deviations from the Plan

The contract:

- **`ANFS_IMAGE_GUEST_EXEC` and `_PUSH` are paths to scripts pmm generates** in the build
  workspace, not literal ssh command lines, so arguments survive the far shell's re-parse. A
  workspace path with whitespace in it is refused (`guest_ssh`): the two are still handed over
  to be used unquoted.
- **`ANFS_IMAGE_WORK`** is added: an empty directory per call, deleted after it. The plan says the
  anfs fingerprint "writes a file" without saying where; it writes there.
- `GUEST_PUSH` copies as root, to an absolute guest path, a directory's contents into that
  directory.
- `ANFS_IMAGE_SSH_PUBKEY` is set on every backend, not only tart.
- The `sh` provisioner leaves `ANFS_IMAGE_DIR` and `ANFS_IMAGE_WORK` out of the script's
  environment as well as the `GUEST` ones: they are host paths. The plan excludes only `GUEST`.
- **A fingerprint must name the step as its first line** (checked with `-ef`), and a path it
  names must exist, or `validate` and the build fail. The plan says "the step first" and leaves
  it there.

The build key:

- It hashes the content of what `fingerprint` names, not the printed paths.
- **The provisioner's own file is not in the key**: editing `sh` rebuilds nothing. One whose
  change is another build prints `$0` in its fingerprint. Documented in CLAUDE.md.
- A symlink in `files/` or in a fingerprinted directory is in the key as `<path> -> <target>`,
  not followed.
- `PMM_IMAGE_BUILD_VERSION` was not bumped: every key changes anyway.

`validate`:

- It runs each step's `fingerprint`, which is how the anfs provisioner refuses
  `sources: sideways` and a key that is not one of `sources`, `repo`, `packages`, `installer`.
- A leftover `anfs:` key in `image.yml` is an error naming the step that replaced it.
- `user.name` on a root image must match `^[a-z_][a-z0-9_-]*$`: `build.sh` and the anfs
  provisioner both put `~<user>` into a root shell. Not in the plan.
- Every regular file in `provision/` is a step, not only names shaped `<NN>-<name>.<ext>`.

The `anfs` provisioner:

- It runs install.sh with `$ANFS_IMAGE_GUEST_EXEC su -l <user> -c 'bash -l -s'`, not a second
  ssh login as the user, so it needs only `EXEC` and `PUSH`. It chowns `~/.local` to the user
  after pushing the sources and sets `XDG_RUNTIME_DIR` in the script (e41c734: `su -l` from a
  session gives none, and core/podman's hook then found no systemd user session).
- `PMM_IMAGE_ANFS_INSTALLER_URL` is gone; the step has an optional `installer:` key. A
  provisioner is its own process and never sees pmm's settings.
- The host-mode fingerprint hashes the names of uncommitted changes (`git status --porcelain`),
  as `anfs_host_fingerprint` did: a further edit to an already dirty file does not change the key.

pmm's own part:

- `user.authorized_keys` is appended by calling the `exec` script directly, as root
  (`cat >> ~user/.ssh/authorized_keys`), by no provisioner. So `build.sh` has one guest call
  between "waiting for ssh" and the shutdown that is not a provisioner's; it goes through the
  same script a provisioner is given, and there is no `ssh` call there.
- `test.sh` and finalize run through the `sh` provisioner, so an `sh` of your own in
  `$PMM_CONFIG_HOME/provisioners` runs those too. `test.sh` sees `ANFS_IMAGE_*` with an empty
  `ANFS_IMAGE_FILES`.
- tart has no finalize step, so the helper and `/tmp/pmm-files` are removed after the last step.
- The build's message for a failing fingerprint is one line, `provision/<step>: <the
  provisioner's stderr>`.
- `prov_run` prefixes a step's output with a bash read loop, so `image build -v` shows it line
  by line (it came in blocks before this plan too).
- `image show` has no "anfs install" line; each step row has a `provisioner` in `--json`, and a
  step nothing provides is marked in the text.

Other:

- pmm has no README, so where an ansible provisioner would go is named in CLAUDE.md and the
  `pmm-images` skill, and the lookup path in `pmm help`.
- One commit for the plan and one for each fix round, not one per step.

## Review and Fixes

The review found the plan's steps and Test Spec all present and one should-fix: `validate` and
the build disagreed about a fingerprint naming a missing path, while CLAUDE.md said validate
caught it. Fixed in 960b5bb (`key_step` is read by both), with the minor findings: "the step
first" enforced, symlinks in a fingerprinted tree, the anfs step quoting `packages:` and refusing
unknown keys, the `user.name` rule, bats tests for `user.authorized_keys` and for `tart_test`
through the provisioner, two test stubs (the ssh stub's failure switch was dead; the tart stub's
`stop` left a 300 s sleep).

Left as they are, and documented: the provisioner's file not being in the key, an `sh` override
taking over finalize and `test.sh`, the whitespace limit on the workspace path.

## Test Results

Final pass, on 960b5bb (core) and 4ffc4a3 (stack) with a clean tree in both worktrees,
macOS 26.6.2 arm64, qemu 11.1.1, bats-core. No failure was found and no code was changed.

bats, each as `bats packages/<tool>/tests </dev/null`:

| Suite | Result | Time |
| --- | --- | --- |
| anfs | 142 of 142 | 46 s |
| pmm | 112 of 112 | 412 s |
| ppm | 28 of 28 | 3 s |
| pcm | 67 of 67 | 36 s |
| psm | 8 of 8 | 3 s |
| wsm | 56 of 56 | 38 s |
| stack ventoy (`ANFS_TEST_LIB` set to the core worktree's lib) | 18 of 18 | 5 s |

431 tests, 0 failures, none skipped. pmm was 97 at the end of plan 02: 14 are `provision.bats`,
one is new in `image-validate.bats`.

`packages/pmm/tests/e2e debian </dev/null`: real qemu on hvf, isolated HOME, the shared ISO
store (read only: the ISO was there). Passed on the first run of this pass, exit 0, 525 s.
Earlier agents report passes on 0d947e5 and 960b5bb. Verbatim:

```
== debian-13: the image
  ok    image validate (1s)
  ok    image show: every file the build reads, and its ISO (0s)
  ok    image list: unbuilt (0s)
  ok    image build --test (183s)
  ok    rebuild is a cache hit (1s)
  ok    image list: built and tested (1s)
  ok    image export -c, a standalone qcow2 (46s)
  ok    image new --from: a child, in the local source (1s)
  ok    a step nothing provides fails validate (0s)
  ok    the child builds on debian-13's build: a .sh step, then the run's own provisioner, in name order; it passes debian-13's test.sh (61s)
  ok    image show web: both images' steps and what runs each; built and tested (2s)
  ok    a machine from the child has both steps' work, and nothing of the build's (29s)
  ok    a file the step's fingerprint names changes the build key (1s)
  ok    image remove is refused while the child is built on it (0s)
  ok    image remove web (0s)
== debian-13: machines
  ok    run --no-start: a, with a read-only share (2s)
  ok    no definition was written (0s)
  ok    the build is held by a (1s)
  ok    up a (25s)
  ok    exec runs a command (0s)
  ok    the share is read-only (0s)
  ok    the profile's one cpu (1s)
  ok    run: a second machine, started (27s)
  ok    ps lists both (1s)
  ok    b has its own disk (0s)
  ok    down a (1s)
  ok    up again: the disk and the settings persisted (26s)
  ok    snapshot, change, reset to it (54s)
  ok    the snapshot holds the build too (0s)
  ok    image remove is refused while held (0s)
  ok    reset onto the build: a new disk (28s)
  ok    list: a and b running (0s)
  ok    snapshot remove (0s)
  ok    run --rm: gone at its shutdown (30s)
  ok    rm a and b (3s)
  ok    the build is no longer held (0s)
  ok    image prune (0s)
  ok    image remove, now that nothing is on it (0s)
  ok    implode (0s)
  ok    implode left nothing
== PASS
```

What the e2e proves of this plan: the `.probe` provisioner in `$PMM_CONFIG_HOME/provisioners`
fails unless all sixteen variables are set, gets 0 from `$ANFS_IMAGE_GUEST_EXEC id -u`, writes a
file with a space in its name through `EXEC` and one through `PUSH`; a machine from the image has
both, the `.sh` step's work (which read `ANFS_IMAGE_NAME`, `_ARCH`, `_USER`, `_FILES`), and no
`/tmp/pmm-files`. Appending to the `.probe` step makes the image `stale`.

## Verification

The plan's list, under an isolated home (the core worktree as the source `core`, stack's as
`stack`, a scratch git repository as `extra`; `PMM_*_HOME`, `PMM_SOCK_DIR` and `TART_HOME` in
temp directories; the Debian ISO read from the real store):

- [x] `lib/pmm/build.sh` has no ssh call between "waiting for ssh" and finalize except through a
  provisioner. Read: after `wait_ssh` there is `guest_ssh_qemu`, `provision_steps` and finalize
  through `prov_run`; the one other guest call is `user.authorized_keys` through the `exec`
  script (Deviations). The shutdown after finalize is `ssh_poweroff`, as before.
- [x] `grep -r 'anfs:' machines/images/` finds nothing; `pmm image build anfs-dev` works. Built
  for real on 960b5bb with `--test`: debian-13 in 167 s, then `50-toolkit.anfs` 172 s, finalize,
  202 s in all, and debian-13's `test.sh` passes (7 of 7). A machine from it has `anfs`, `ppm`
  and `pmm`; `anfs src list` shows `stack`, `core` and `extra` as its own local sources, in the
  host's order; the sources and `~/.local` are owned by `pmm`; `podman.socket` is enabled for
  the user; there is no `/tmp/pmm-files`.
- [x] The anfs step in host mode, for real: after a commit in the source `extra`,
  `image list` shows `anfs-dev` as `stale`.
- [x] `packages/pmm/tests/e2e debian` (above).
- [ ] The macOS image (`ppm vm build macos`): **not run.** See below.
- [x] `packages/anfs/tests/bash.bats` covers `provisioners/*` as entry points (the "every entry
  point sources the floor" test lists `packages/pmm/home/.local/lib/pmm/provisioners/*`; both
  files are mode 755).

Also by hand:

- `image validate`: the four shipped images, 0 errors, 0 warnings. `image show anfs-dev` lists
  `10-base.sh`, `50-toolkit.anfs` and finalize; `--json` gives `sh`, `anfs`, `sh` as their
  provisioners. `image show anfs-test-macos` prints its one step and no finalize.
- On a child image, each of these fails `validate` (exit 1) with the text quoted: a `.yml` step
  ("nothing provides .yml steps: install the package that does"); a file with no extension; a
  `yml` provisioner whose `check` prints "ansible is not installed"; one whose fingerprint names
  `/nonexistent/x` (and `image build` refuses with the same line); an `.anfs` step with
  `sources: sideways`; one with `source: host`; an `anfs:` key in `image.yml`.
- **`user.authorized_keys`, for real** (stubbed only until now): a root image copied from
  debian-13 with one extra key built in 164 s; a machine from it has two lines in
  `~/.ssh/authorized_keys` (owner `pmm`, mode 600), and an ssh login with the extra key works
  and has passwordless sudo.
- `pmm help` has the lookup path, the three verbs and the variable table.
- `grep -rn PIM_` over both worktrees finds only `provision.bats`'s own check.

Afterwards: no qemu is running, `/tmp/pmm-501` has kept its mtime, the real ISO store is as it
was (`ls -la` diffed), no `/tmp/pmm-e2e.*` or `pmm-step.*` is left, `/tmp/pmm-bats-501` is empty,
both worktrees are clean. The tart machine that was running on this host (a `roundtrip-macos`
box, pid 40541) is still running and was not touched.

## Not Verified on This Machine

- **tart, for real**: no tart build or test ran, so `ppm vm build macos` in the plan's
  Verification list is open. `_tart_guest`, `_tart_guest_close`, the askpass path, `files/` on
  tart and `tart_test` through the provisioner are covered by stubbed bats tests only.
  `anfs-test-macos/provision/10-provision.sh` with its renamed variable (`ANFS_IMAGE_SSH_PUBKEY`)
  has not run in a guest. A tart machine was running on this host and was left alone.
- **Linux and kvm**: nothing ran there.
- **amd64**: every real run is arm64.
- **`e2e fedora`**: not run in this plan (it passed at the end of plan 02; `fedora-44` did not
  change since, the build path under it did).
- **The anfs step's `sources: pushed`, `repo:`, `packages:` and `installer:`** in a guest: bats
  checks the generated script only. The real builds were host mode.
- **The host's sources as git URLs** in host mode: in the real build all three were local paths,
  so the guest's `user.list` held paths. The URL branch is bats only.
- **`packages/anfs/tests/roundtrip`** and **`roundtrip-macos`**: not run.
- **A real install** of this branch (`ppm install core/anfs`), and a provisioner stowed into
  `~/.local/lib/pmm/provisioners/` by a package: the fixture and e2e provisioners are in
  `$PMM_CONFIG_HOME/provisioners`. The unit's completion criterion "a provisioner stowed by a test
  fixture package runs a step" is met only in that weaker form.
- No ansible (`yml`) provisioner exists, as the plan says, so nothing has tried
  `ANFS_IMAGE_GUEST_HOST`, `_PORT`, `_USER`, `_KEY` as a connection of its own; the e2e checks
  only that they are set.
- A provisioner under bash 3.2 re-executing through `bash.sh`: `bash.bats` checks that the source
  line is there, not a run under `/bin/bash`.
- The `pmm-images` skill's "Writing a provisioner" has not been used to write one.

## Left Open

- An `sh` override in `$PMM_CONFIG_HOME/provisioners` takes over finalize and `test.sh`.
- A build workspace path with whitespace is refused.
- The host-mode fingerprint does not see a further edit to an already dirty file.
- The old `PIM_*` names fail silently in a step of someone's own (unset, no warning).
- Moving the provisioners to `lib/anfs/provisioners/` is a later unit's, when pcm builds with them.
- From the earlier plans, unchanged: the two round trips, tart for real, `state.yml`'s unread
  `base`, the tart snapshot name clash, and plan 02's list.
