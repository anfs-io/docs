---
---

# Plan 04 — Boxes: a stack applied to a container or a VM

Needs plans 01 and 03. Replaces `dev-source/03-boxes`.

## Context — read these files first

- `units/stacks.md` ("Targets"); plans 01 and 03 and their logs
- core `packages/pcm` (`exec`, `validate.sh`'s `check_volumes` and `x-pcm.allow_mounts`), `pmm`
  (`exec`; the `anfs` provisioner in `lib/pmm/provisioners/anfs`)
- anfs-dev `packages/dev/home/.local/lib/ppm/{container,vm}.sh`, `containers/anfs-test-*`,
  `machines/anfs-test-macos`, `packages/dev/tests/{harness,roundtrip}` (how the sources are put
  into a box today)
- anfs-dev `packages/dev/home/.local/bin/anfs-work` (jobs, worktrees)

## Implementation

### 1. A box, the native way

- A container box is a pcm container whose Containerfile runs
  `RUN <install.sh> && anfs install <stack> --build` and whose start runs `anfs install <stack>`
  (the rest: spaces, vaults, statuses). `compose.yml` holds the mounts. No anfs key in compose.
- A VM box is a pmm image whose `anfs` provision step names a stack: `stack: lgat/rails` beside
  (and in time instead of) `packages:`.
- Rebuilds follow what went in: pcm's build key and pmm's fingerprint include the stack's
  manifest and the manifests of the stacks it depends on.

### 2. `-t/--target`

`anfs install|remove <stack> -t container:<name>` and `-t machine:<name>` run anfs on the box
through `pcm exec` / `pmm exec`. When anfs is not on it, `install` first runs `install.sh` there
with the host's sources available (section 3), then the stack.

### 3. Mount roots

pcm validate allows these host paths to be mounted, beyond its own:

| Root | Mode |
| --- | --- |
| `$WSM_SPACES_HOME` (spaces) | read-write |
| `~/.local/share/anfs/sources` | read-only |
| `$ANFS_WORK_HOME` (`~/work`) | read-write |

Anything else stays refused unless `x-pcm.allow_mounts` names it. **They are mounted at the
same absolute path inside the box** as on the host: a git worktree's `.git` names its main repo
by absolute path, so jobs only work in a box if both sources and `~/work` are where the host has
them. Rootless podman with `--userns=keep-id`, so files written in a mount are the person's.

### 4. The anfs-dev box

- anfs-dev `stacks/dev/stack.yml` (the toolkit and the dev package) and a container box per
  distro built from it (`containers/anfs-test-debian`, `-fedora` become boxes with a
  `container.yml`), the sources mounted read-only and a job's worktrees read-write.
- `ppm container` and `ppm vm` (anfs-dev `container.sh`, `vm.sh`) go: `pcm up|exec` and
  `pmm up|exec` on the box, and `anfs install -t`. `ppm user` goes to anfs if anything still
  needs it (check), else goes.
- The box writes nothing into the mounted sources: its state is its own HOME's.

## Test Spec

- `packages/anfs/tests`: `-t container:x` runs `pcm exec x -- anfs install ...` (stubbed), and
  bootstraps first when `anfs` is missing on the box; `-t machine:x` the same through pmm.
- pcm and pmm suites are off: their tests (mount roots, the `stack:` key in the `anfs` step,
  the fingerprint) are written and listed as not run.

## Verification

- [ ] `packages/anfs/tests/all`; anfs-dev `bats packages/dev/tests`
- [ ] Deferred (needs podman / a VM): build the debian box from `anfs-dev/dev`, run a job's
      bats inside it, `git status` in a mounted worktree works inside
