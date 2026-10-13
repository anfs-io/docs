---
---

# Plan 03 — Trying a job in a box: `anfs work install|shell|reset`

## Context — read these files first

- `units/dev-source.md`; plan 02 and its `log.md` (`anfs work`, the job's number, `env`)
- `anfs-dev` `packages/dev/home/.local/lib/ppm/{container,vm,user}.sh` — what is replaced
- `container-images/01-run-exec-snapshot` and its `log.md` — `pcm run --name`, `pcm exec`,
  `pcm snapshot`
- `machine-images/01-fold-pim-into-pmm/log.md` — `pmm run --name`, `pmm exec`, `pmm snapshot`
- `anfs-dev` `containers/anfs-test-*`, `machines/images/anfs-test-macos` — the boxes' contract:
  users `owner` (sudo with a password) and `other` (none), the sources read-only at
  `/src/<alias>`

## Overview

`ppm container` and `ppm vm` are twenty commands. Eighteen of them are one line each: `pcm` or
`pmm` with `anfs-test-` put before the name. They were written when ppm was the only tool, and
pcm and pmm now have every one of those verbs themselves. The two that do something are the
`install` commands: they point the box's `user.list` at the mounted working tree and run
`install.sh` there as a test user, so the box tests the tree and not what is pushed.
`ppm user` does the same on this host for a throwaway user.

After this plan that one thing is `anfs work install`, it tests a job's trees, and two jobs can
each have their own box. Everything else about a box is `pcm` or `pmm` with the box's name.
`container.sh`, `vm.sh` and `user.sh` are deleted.

## Implementation

### 1. Commands

```
anfs work install <job> <box> [--as owner|other] [--pushed] [-- <install.sh args>]
anfs work shell   <job> <box> [--as owner|other]
anfs work reset   <job> <box>
```

| Command | Does |
| --- | --- |
| `install` | Makes the job's box if it is not there, starts it, lists the job's trees in the test user's `user.list` (a source the job does not hold: the installed one), and runs `install.sh` from the job's core as that user. `--pushed` runs the published installer instead and uses no tree: what a new user gets |
| `shell` | A login shell in the box as the test user |
| `reset` | Removes the box, so the next `install` starts from a fresh one |

### 2. Boxes

| Box | Is | Made with | Named |
| --- | --- | --- | --- |
| `debian`, `fedora` | a container | `pcm run anfs-test-<distro> --name ...`, the job's directory read-only at `/src` | `work-<n>-<distro>` |
| `macos` | a tart VM (Apple silicon) | `pmm run anfs-test-macos --name ... --share` | `work-<n>-macos` |
| `user` | another user on this host (what `ppm user` made): the non-owner Homebrew path | `sysadminctl` / `useradd`, with sudo | `anfswork<n>` |
| `home` | you, with a temp `HOME` whose tools are the job's: its packages linked with no hooks and no declared software | `mktemp`, the linker | under the job's `TMPDIR` |

- `<n>` is the job's number, so two jobs' boxes never share a name. `anfs work show` lists a
  job's boxes with their state; `anfs work remove` removes them with the job.
- `home` has no `install` (nothing to install: Homebrew is the host's and is not touched) and
  no `--as`. It is the cheapest way to run a job's `pmm` or `pcm` by hand.
- `macos` takes the tart lock of plan 02.
- A snapshot, the box's address, stopping it, its logs: `pcm` and `pmm` with the box's name.
  Building the macOS image is `pmm image build anfs-test-macos`.

### 3. What goes

`container.sh`, `vm.sh`, `user.sh` and their completions; `PPM_CONTAINER_INSTALLER_URL` and
`PPM_VM_INSTALLER_URL` become one `ANFS_WORK_INSTALLER_URL`. With `hooks.sh` gone in plan 02
and `move.sh` in `linker/03`, the `dev` package stows nothing into `lib/ppm`.

The round trips and the harness are not changed: they build their boxes with podman alone, so
that a broken pcm cannot break what tests it.

### 4. Documentation

`anfs-dev`'s CLAUDE.md "Testing"; core's CLAUDE.md loses `ppm container`, `ppm vm`, `ppm user`
and the sentence that a `cmd_` function in `lib/ppm` becomes a command.

## Test Spec

`anfs-dev` `packages/dev/tests/boxes.bats` (pcm, pmm and the user tools stubbed):

- `install <job> debian`: `pcm run` is called once with the job's name for the box and the job's
  directory at `/src`; a second `install` does not make another box.
- The `user.list` written in the box lists the job's trees first and core last; a source the
  job lacks is the installed one's path.
- `--as other`, `--pushed` (no tree is linked; refused when the box has linked trees), and
  arguments after `--` reach `install.sh`.
- Two jobs: two box names. `reset` removes only its job's box. `anfs work remove` removes the
  job's boxes.
- `home`: `shell` runs with `HOME` under the job's `TMPDIR` and the job's `pmm` first on `PATH`;
  `install` is refused (64).

## Verification

- [ ] `bats packages/dev/tests </dev/null`
- [ ] `anfs work install <job> debian`, then `anfs work shell <job> debian`: `ppm --version` in
      the box is the job's
- [ ] The same for `macos` on this Mac, and for `user`
- [ ] `grep -rn 'ppm container\|ppm vm\|ppm user'` in core, stack and `anfs-dev` finds nothing
